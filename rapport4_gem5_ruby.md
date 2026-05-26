# Rapport 4 — Documentation gem5/Ruby
## Ciblée sur la modélisation de la liste des copies (stage MADMAX/TIMA)

---

## Vue d'ensemble de l'architecture Ruby

Ruby est le sous-système mémoire détaillé de gem5. Il remplace les caches "classiques" de gem5 par un modèle cycle-précis qui simule : hiérarchies de caches, protocoles de cohérence, contrôleurs DMA et mémoire, et un réseau d'interconnexion (Garnet).

```
┌──────────────────────────────────────────────────────────┐
│                        gem5 CPU                          │
│                        (RubyPort)                        │
└────────────────────────┬─────────────────────────────────┘
                         │ gem5 Packet
                         ▼
┌──────────────────────────────────────────────────────────┐
│                      Sequencer                           │
│   (convertit Packet → RubyRequest → mandatoryQueue)      │
└────────────────────────┬─────────────────────────────────┘
                         │ RubyRequest
                         ▼
┌──────────────────────────────────────────────────────────┐
│           Contrôleur L1Cache (généré par SLICC)          │
│              Machine à états — Protocol FSM              │
└────────────┬──────────────────────────────┬──────────────┘
             │ MessageBuffer (vnet 0,1,2)    │
             ▼                              ▼
┌────────────────────────────────────────────────────────┐
│                   Réseau Ruby (Garnet)                  │
│        Routers — Links — NetworkInterfaces             │
└────────────┬──────────────────────────────┬────────────┘
             │                              │
             ▼                              ▼
┌────────────────────────┐    ┌─────────────────────────┐
│  Contrôleur Directory  │    │  Contrôleur L1Cache (N) │
│  (généré par SLICC)    │    │                         │
│  + DirectoryMemory     │    └─────────────────────────┘
└───────────┬────────────┘
            │ requestToMemory / responseFromMemory
            ▼
┌──────────────────────────┐
│   MemCtrl (DRAM)         │
└──────────────────────────┘
```

**Fichiers clés :**
- `src/mem/ruby/system/RubySystem.{hh,cc,py}` — point d'entrée
- `src/mem/ruby/system/Sequencer.{hh,cc}` — interface CPU ↔ Ruby
- `src/mem/ruby/structures/CacheMemory.{hh,cc}` — structure cache générique
- `src/mem/ruby/structures/DirectoryMemory.{hh,cc}` — structure directory
- `src/mem/ruby/slicc_interface/AbstractController.hh` — classe de base de tous les contrôleurs
- `src/mem/ruby/network/garnet/` — implémentation Garnet NoC
- `src/mem/ruby/protocol/` — tous les protocoles SLICC

---

## SLICC — Specification Language for Implementing Cache Coherence

SLICC est le DSL (Domain Specific Language) de Ruby pour décrire les protocoles de cohérence. Les fichiers `.sm` (state machine) sont compilés par le compilateur SLICC (Python) en C++ avant la compilation SCons.

### Structure d'un fichier `.sm`

Un fichier SLICC suit toujours cet ordre :

```
1. machine(MachineType:NOM, "description")
   : <paramètres SimObject>
   : <MessageBuffer declarations>
{
2.  state_declaration(State, ...)  // états stables + transitoires
3.  enumeration(Event, ...)        // événements
4.  structure(Entry, ...)          // structure d'une entrée de cache/directory
5.  structure(TBE, ...)            // transaction buffer entry (MSHR-like)
6.  structure(TBETable, external)  // table des TBEs
7.  // Déclarations de fonctions AbstractController
8.  // Définitions : getState, setState, getAccessPermission, ...
9.  out_port(...)                  // ports de sortie (envoi de messages)
10. in_port(...)                   // ports d'entrée (réception + trigger)
11. action(...)                    // actions atomiques
12. transition(...)                // table de transitions
}
```

### Syntaxe essentielle

**Déclaration machine :**
```cpp
machine(MachineType:Directory, "Mon directory")
  : DirectoryMemory * directory;
    Cycles latency := 6;
    MessageBuffer * requestToDir, network="From", virtual_network="0",
          vnet_type="request";
    MessageBuffer * responseFromDir, network="To", virtual_network="1",
          vnet_type="response";
    MessageBuffer * requestToMemory;
    MessageBuffer * responseFromMemory;
{
    // corps
}
```

**États :**
```cpp
state_declaration(State, desc="États du directory",
                  default="Directory_State_I") {
    I,    AccessPermission:Read_Write,  desc="Invalide dans tous les caches";
    S,    AccessPermission:Read_Only,   desc="Partagé";
    M,    AccessPermission:Invalid,     desc="Modifié dans un cache";
    // États transitoires
    S_m,  AccessPermission:Busy,        desc="Attend réponse mémoire pour S";
}
```

**Permissions** : `Invalid`, `NotPresent`, `Busy`, `Read_Only`, `Read_Write`. Servent aux accès fonctionnels (debug).

**Événements :**
```cpp
enumeration(Event, desc="Events du directory") {
    GetS,     desc="Requête read-only d'un cache";
    GetM,     desc="Requête exclusive d'un cache";
    PutS,     desc="Writeback shared";
    PutM,     desc="Writeback modified";
    MemData,  desc="Données arrivant de la mémoire";
    MemAck,   desc="Ack de la mémoire après WB";
}
```

**Structure d'entrée directory :**
```cpp
structure(Entry, desc="Entrée du directory",
          interface="AbstractCacheEntry", main="false") {
    State   DirState;
    NetDest Sharers;    // bitvector de tous les sharers
    NetDest Owner;      // propriétaire en état M
    // Pour Ackwise/DCC : ajouter ici vos champs personnalisés
    // ex: int NumSharers; bool Overflow; ...
}
```

**`main="false"`** : indique à la replacement policy d'ignorer cette entrée (essentiel pour le directory).

**TBE (Transaction Buffer Entry) :**
```cpp
structure(TBE, desc="TBE pour requêtes en transit") {
    Addr    PhysicalAddress;
    State   TBEState;
    DataBlock DataBlk;
    int     AcksOutstanding, default=0;
    MachineID Requestor;
}

structure(TBETable, external="yes") {
    TBE  lookup(Addr);
    void allocate(Addr);
    void deallocate(Addr);
    bool isPresent(Addr);
}

TBETable TBEs, template="<Directory_TBE>", constructor="m_number_of_TBEs";
```

**Ports d'entrée/sortie :**
```cpp
out_port(response_out, ResponseMsg, responseFromDir);
out_port(forward_out,  RequestMsg,  forwardToCache);
out_port(memQueue_out, MemoryMsg,   requestToMemory);

in_port(request_in, RequestMsg, requestToDir) {
    if (request_in.isReady(clockEdge())) {
        peek(request_in, RequestMsg) {
            // in_msg est disponible ici
            if (in_msg.Type == CoherenceRequestType:GetS) {
                trigger(Event:GetS, in_msg.addr);
            }
        }
    }
}
```

**Actions :**
```cpp
action(sendDataToReq, "d", desc="Envoyer données au demandeur") {
    peek(memQueue_in, MemoryMsg) {
        enqueue(response_out, ResponseMsg, 1) {
            out_msg.addr        := address;
            out_msg.Type        := CoherenceResponseType:Data;
            out_msg.Sender      := machineID;
            out_msg.Destination.add(in_msg.OriginalRequestorMachId);
            out_msg.DataBlk     := in_msg.DataBlk;
            out_msg.MessageSize := MessageSizeType:Response_Data;
        }
    }
}
```

- `enqueue(port, TypeMsg, latence)` — envoie un message après `latence` cycles
- `peek(port, TypeMsg)` — inspecte le message en tête de file (déclare `in_msg`)
- `address` — variable implicite = adresse du bloc traité dans la transition

**Transitions :**
```cpp
// (état_initial, événement, état_final)
transition(I, GetS, S_m) {
    sendMemRead;      // action : demander données à la mémoire
    addReqToSharers;  // action : ajouter demandeur aux sharers
    popRequestQueue;  // action : consommer le message entrant
}

// Plusieurs états initiaux / événements possibles :
transition({I, S}, {PutSNotLast, PutMNonOwner}) {
    sendPutAck;
    popRequestQueue;
}
```

**Gestion des conflits (stall) :**
```cpp
// Option 1 : stall simple (bloque tout le port d'entrée)
transition({S_m, M_m}, GetS) {
    z_stall;  // action prédéfinie
}

// Option 2 : recycler (meilleure perf mais irréaliste)
action(z_recycleMandatoryQueue, "z", desc="Recycler la queue") {
    mandatoryQueue_in.recycle(clockEdge(), cyclesToTicks(recycle_latency));
}

// Option 3 : stall_and_wait (plus réaliste, nécessite un wakeup explicite)
action(z_stallAndWait, "z", desc="Stall and wait") {
    stall_and_wait(request_in, address);
}
// + action de réveil dans les transitions qui passent en état stable
action(kd_wakeUp, "kd", desc="Réveiller les dépendants") {
    wakeUpBuffers(address);
}
```

---

## DirectoryMemory — Structure C++ sous-jacente

`DirectoryMemory` (`src/mem/ruby/structures/DirectoryMemory.{hh,cc}`) est un tableau de pointeurs vers `AbstractCacheEntry*`. Il couvre **toute** la mémoire physique, mais les entrées sont allouées **paresseusement**.

```cpp
// Initialisation : tableau de m_num_entries pointeurs NULL
m_num_entries = m_size_bytes / m_block_size;
m_entries = new AbstractCacheEntry*[m_num_entries];

// Lookup / allocation depuis SLICC
Entry getDirectoryEntry(Addr addr), return_by_pointer="yes" {
    Entry dir_entry := static_cast(Entry, "pointer", directory[addr]);
    if (is_invalid(dir_entry)) {
        dir_entry := static_cast(Entry, "pointer",
                                 directory.allocate(addr, new Entry));
    }
    return dir_entry;
}
```

**Points importants pour le stage :**
- Le directory actuel est de **taille illimitée** (une entrée par ligne de cache de toute la mémoire). L'objectif du stage est de le **limiter**.
- Pour modéliser un directory à taille limitée, il faut modifier `DirectoryMemory` pour que `m_num_entries` soit borné, et gérer les **remplacements d'entrées** (éviction du directory).
- Alternativement, on peut implémenter la logique de limitation directement dans le fichier `.sm` en utilisant une table de taille fixe (comme une `CacheMemory` avec replacement policy).

**`NetDest`** (type SLICC pour la sharelist) :
- C'est un bitvector pleine taille sur tous les nœuds — exactement le problème O(n) à résoudre.
- Pour Ackwise/DCC, vous remplacerez `NetDest Sharers` par une structure compacte (pointeurs limités + bit de débordement, etc.).

---

## Protocoles existants — Guide de lecture

### MI_example — le plus simple

| Fichier | Rôle |
|---------|------|
| `MI_example.slicc` | Déclare le protocole et inclut les `.sm` |
| `MI_example-msg.sm` | Types de messages (GETX, GETS, PUTX, ACK, DATA…) |
| `MI_example-cache.sm` | Contrôleur L1 cache (états M, I + transitoires) |
| `MI_example-dir.sm` | Contrôleur directory (états I, M + transitoires DMA) |
| `MI_example-dma.sm` | Contrôleur DMA |

Le directory MI_example a :
- **États stables** : `I` (données en mémoire, aucun cache), `M` (données dans un seul cache)
- **Entrée** : `{ State DirectoryState; NetDest Sharers; NetDest Owner; }`
- **Transitions** clés :
  - `I + GETX → IM` : fetch mémoire, noter le futur propriétaire
  - `IM + Memory_Data → M` : envoyer données au demandeur
  - `M + GETX → M` : forward la requête à l'owner courant, changer l'owner
  - `M + PUTX → MI` : writeback vers mémoire, envoyer ack

### MESI_Two_Level — protocole de référence

Plus complet avec états S (shared), M, E, I. Le directory entry ne stocke qu'un `Owner` (pas de sharers — protocole simplifié). Utile pour comprendre comment implémenter un protocole MESI complet.

### MOESI_CMP_directory — le plus pertinent pour le stage

C'est le seul protocole qui a une véritable **sharelist** (`NetDest Sharers`) utilisée en production dans gem5. Étudier ce protocole pour comprendre la gestion des invalidations multicast.

---

## Créer un nouveau protocole — Procédure pas à pas

### 1. Créer le répertoire du protocole

```
src/mem/ruby/protocol/
├── MON_PROTOCOL.slicc         ← fichier principal
├── MON_PROTOCOL-msg.sm        ← types de messages
├── MON_PROTOCOL-cache.sm      ← contrôleur L1 cache
├── MON_PROTOCOL-dir.sm        ← contrôleur directory (à modifier)
├── MON_PROTOCOL-dma.sm        ← contrôleur DMA (peut copier MI_example)
└── SConsopts                  ← enregistrement SCons
```

### 2. SConsopts

```python
Import('*')
main.Append(ALL_PROTOCOLS=['MON_PROTOCOL'])
main.Append(PROTOCOL_DIRS=[Dir('.')])
```

### 3. Fichier .slicc

```cpp
protocol "MON_PROTOCOL";
include "RubySlicc_interfaces.slicc";  // types de base Ruby
include "MON_PROTOCOL-msg.sm";
include "MON_PROTOCOL-cache.sm";
include "MON_PROTOCOL-dir.sm";
include "MON_PROTOCOL-dma.sm";
```

### 4. Compiler

```bash
# gem5 >= 23.1 (Kconfig)
scons defconfig build/X86_MON build_opts/X86
scons setconfig build/X86_MON RUBY_PROTOCOL_MON_PROTOCOL=y SLICC_HTML=y
scons build/X86_MON/gem5.opt -j$(nproc)

# gem5 <= 23.0 (SCons direct)
scons build/X86_MON/gem5.opt --default=X86 PROTOCOL=MON_PROTOCOL SLICC_HTML=True -j$(nproc)
```

`SLICC_HTML=y` génère des tableaux HTML des transitions dans `build/X86_MON/mem/protocol/html/` — très utile pour déboguer.

### 5. Configuration Python

```python
# msi_caches.py (à adapter)
from m5.objects import *

class MyDirectory(Directory_Controller):  # généré par SLICC
    _version = 0
    @classmethod
    def versionCount(cls):
        cls._version += 1
        return cls._version - 1

    def __init__(self, ruby_system, ranges, mem_ctrls):
        super().__init__()
        self.version = self.versionCount()
        self.addr_ranges = ranges
        self.ruby_system = ruby_system
        self.directory = RubyDirectoryMemory()
        self.memory = mem_ctrls[0].port
        self.connectQueues(ruby_system)

    def connectQueues(self, ruby_system):
        self.requestFromCache = MessageBuffer(ordered=True)
        self.requestFromCache.slave = ruby_system.network.master
        # ... autres buffers ...
        self.responseFromMemory = MessageBuffer()
```

---

## Garnet — Réseau sur puce (NoC)

Garnet 3.0 est l'implémentation NoC cycle-précise de gem5, dans `src/mem/ruby/network/garnet/`.

### Architecture

```
┌──────────────────────────────────────────────────────┐
│  Contrôleur (Cache/Directory)                        │
│  MessageBuffer ──→ NetworkInterface (NI)             │
│                    ↓ flit injection                  │
│              Router ──→ Router ──→ Router            │
│              ↓ flit extraction                       │
│  NetworkInterface (NI) ──→ MessageBuffer             │
│  Contrôleur destination                              │
└──────────────────────────────────────────────────────┘
```

### Composants clés

| Composant | Fichier | Rôle |
|-----------|---------|------|
| `GarnetNetwork` | `GarnetNetwork.{hh,cc}` | Instancie routers, links, NIs |
| `NetworkInterface` | `NetworkInterface.{hh,cc}` | Convertit messages ↔ flits |
| `Router` | `Router.{hh,cc}` | Switch Allocator + Crossbar |
| `InputUnit` | `InputUnit.{hh,cc}` | Buffer d'entrée par VC |
| `OutputUnit` | `OutputUnit.{hh,cc}` | Crédits vers routeur downstream |
| `SwitchAllocator` | `SwitchAllocator.{hh,cc}` | Arbitre VC |
| `flit` | `flit.{hh,cc}` | Unité de transfert |

### Paramètres importants

```python
# Activer Garnet depuis la ligne de commande
./build/X86_MON/gem5.opt configs/... --network=garnet --topology=Mesh_XY --mesh-rows=4

# Paramètres GarnetNetwork.py
ni_flit_size = 16       # taille du flit en octets (default 128 bits)
vcs_per_vnet = 4        # VCs par réseau virtuel
buffers_per_data_vc = 4 # flits buffer par VC data
buffers_per_ctrl_vc = 1 # flits buffer par VC contrôle
routing_algorithm = 0   # 0=table, 1=XY, 2=custom
```

### Flux d'un message de cohérence dans Garnet

1. Le contrôleur SLICC fait `enqueue(out_port, Msg, latence)` → message dans un `MessageBuffer`
2. Le `NetworkInterface` lit le `MessageBuffer`, découpe le message en **flits** (contrôle = 1 flit, données = 5 flits par défaut)
3. Les flits traversent les routers selon la table de routage (XY par défaut dans un mesh)
4. Le NI destination reçoit les flits, reconstruit le message, le met dans le `MessageBuffer` de destination
5. Le contrôleur destination consomme le message via `in_port`

### Topologies disponibles

| Topologie | Description |
|-----------|-------------|
| `Mesh_XY` | mesh 2D avec routage XY (le plus courant) |
| `MeshDirCorners_XY` | directories aux coins du mesh |
| `Crossbar` | crossbar centralisé |
| `Pt2Pt` | point à point |
| `CrossbarGarnet` | crossbar dans Garnet |

```bash
# Exemple de configuration Mesh 4x4 avec 16 cœurs et 16 directories
./build/X86_MON/gem5.opt ... \
    --num-cpus=16 --num-dirs=16 \
    --network=garnet --topology=Mesh_XY --mesh-rows=4
```

---

## Modéliser un directory à taille limitée — Points d'entrée concrets

### Approche 1 : Modifier `DirectoryMemory` (niveau C++)

Limiter `m_num_entries` à une valeur fixe et implémenter une politique de remplacement :

```cpp
// DirectoryMemory.hh : ajouter un paramètre max_entries
uint64_t m_max_entries;  // nouveau champ

// Dans init() : m_num_entries = min(m_size_bytes / m_block_size, m_max_entries)
// Implémenter allocate() avec politique de remplacement (LRU, etc.)
```

### Approche 2 : Réutiliser `CacheMemory` pour le directory (recommandée)

`CacheMemory` a déjà toutes les politiques de remplacement. On peut l'utiliser comme directory cache :

```cpp
// Dans le .sm
machine(MachineType:Directory, "Mon directory à taille limitée")
  : CacheMemory * dir_cache;      // remplace DirectoryMemory
    Cycles latency := 6;
    int num_tags_per_entry := 4;  // paramètre Ackwise-like
    ...
```

### Approche 3 : Implémenter la logique dans SLICC

Modéliser un directory cache avec overflow en SLICC pur :
- Ajouter un état `OVERFLOW` dans la sharelist
- Quand la sharelist déborde sa capacité, passer en `OVERFLOW` → broadcast/multicast élargi

### Modifications nécessaires dans le fichier `-dir.sm`

```cpp
// Exemple d'entrée directory modifiée pour sharelist compacte
structure(Entry, desc="Entrée directory compacte",
          interface="AbstractCacheEntry", main="false") {
    State   DirState;
    // Remplace NetDest Sharers par une structure compacte :
    int     NumSharers;        // compteur de copies
    NetDest LimitedSharers;   // k pointeurs (k << N)
    bool    OverflowBit;       // débordement → multicast élargi
    NetDest Owner;
}
```

---

## Fichiers types Ruby/SLICC à lire dans cet ordre

1. **`MI_example-msg.sm`** — comprendre les types de messages de base
2. **`MI_example-dir.sm`** — le directory le plus simple (états I, M seulement)
3. **`MESI_Two_Level-dir.sm`** — directory MESI complet mais sans sharers
4. **`MI_example-cache.sm`** — cache controller de référence
5. **`src/learning_gem5/part3/MSI-dir.sm`** — tutorial directory MSI avec Sharers et Owner

---

## Commandes de débogage utiles

```bash
# Activer les traces Ruby
./gem5.opt ... --debug-flags=RubySlicc,RubyNetwork

# Activer les traces Garnet
./gem5.opt ... --debug-flags=RubyNetwork,GarnetSynFlit

# Générer les tableaux HTML des transitions SLICC
# (avec SLICC_HTML=y à la compilation)
# → build/X86_MON/mem/protocol/html/
```
