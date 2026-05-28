# Plan d'implémentation — Directory à taille limitée avec sharelist compacte
## Guide fichier par fichier pour le stage MADMAX/TIMA

---

## Table des matières

1. [Vue d'ensemble des phases](#1-vue-densemble-des-phases)
2. [Arborescence des fichiers concernés](#2-arborescence-des-fichiers-concernés)
3. [Phase 1 — Création du nouveau protocole (copie)](#3-phase-1--création-du-nouveau-protocole)
4. [Phase 2 — Directory à taille limitée](#4-phase-2--directory-à-taille-limitée)
5. [Phase 3 — Sharelist compacte (Ackwise)](#5-phase-3--sharelist-compacte-ackwise)
6. [Phase 4 — Configuration Python et compilation](#6-phase-4--configuration-python-et-compilation)
7. [Phase 5 — Tests et validation](#7-phase-5--tests-et-validation)
8. [Dépendances entre les modifications](#8-dépendances-entre-les-modifications)

---

## 1. Vue d'ensemble des phases

```
Phase 1 : Copier MOESI_CMP_directory → MOESI_CMP_limited_dir
          (Nouveau répertoire, fichiers identiques, proto fonctionnel)
              │
              ▼
Phase 2 : Rendre le directory de taille limitée
          (Remplacer DirectoryMemory par CacheMemory + mécanisme de recall)
              │
              ▼
Phase 3 : Remplacer NetDest Sharers par une sharelist compacte
          (K pointeurs + overflow bit + compteur → Ackwise simplifié)
              │
              ▼
Phase 4 : Configuration Python + compilation SCons/Kconfig
              │
              ▼
Phase 5 : Tests de validation fonctionnelle + benchmarks
```

---

## 2. Arborescence des fichiers concernés

```
gem5/
├── src/mem/ruby/
│   ├── protocol/
│   │   ├── MOESI_CMP_directory.slicc          ← ORIGINAL (ne pas toucher)
│   │   ├── MOESI_CMP_directory-dir.sm         ← ORIGINAL
│   │   ├── MOESI_CMP_directory-L1cache.sm     ← ORIGINAL
│   │   ├── MOESI_CMP_directory-L2cache.sm     ← ORIGINAL
│   │   ├── MOESI_CMP_directory-msg.sm         ← ORIGINAL
│   │   ├── MOESI_CMP_directory-dma.sm         ← ORIGINAL
│   │   │
│   │   ├── MOESI_CMP_limited_dir.slicc        ← À CRÉER (Phase 1)
│   │   ├── MOESI_CMP_limited_dir-dir.sm       ← À CRÉER (Phase 1, puis modifier Phases 2+3)
│   │   ├── MOESI_CMP_limited_dir-L1cache.sm   ← À CRÉER (copie, peu de modifs)
│   │   ├── MOESI_CMP_limited_dir-L2cache.sm   ← À CRÉER (copie, peu de modifs)
│   │   ├── MOESI_CMP_limited_dir-msg.sm       ← À CRÉER (Phase 1, puis modifier Phase 2)
│   │   ├── MOESI_CMP_limited_dir-dma.sm       ← À CRÉER (copie directe)
│   │   └── SConsopts                          ← À MODIFIER (ajouter le protocole)
│   │
│   └── structures/
│       ├── DirectoryMemory.hh                 ← À LIRE (comprendre l'interface)
│       ├── DirectoryMemory.cc                 ← À LIRE
│       ├── DirectoryMemory.py                 ← À LIRE
│       ├── CacheMemory.hh                     ← À LIRE (interface à utiliser)
│       └── CacheMemory.cc                     ← À LIRE
│
├── configs/ruby/
│   ├── MOESI_CMP_directory.py                 ← ORIGINAL (ne pas toucher)
│   ├── MOESI_CMP_limited_dir.py              ← À CRÉER (Phase 4)
│   └── Ruby.py                                ← À LIRE (create_directories)
│
└── build_opts/ ou Kconfig                     ← À MODIFIER (Phase 4)
```

---

## 3. Phase 1 — Création du nouveau protocole

### 3.1 Fichier : `src/mem/ruby/protocol/MOESI_CMP_limited_dir.slicc`

**Action :** Créer ce fichier.

**Contenu :** Identique à `MOESI_CMP_directory.slicc` mais avec les noms modifiés.

```slicc
protocol "MOESI_CMP_limited_dir";
include "MOESI_CMP_limited_dir-msg.sm";
include "MOESI_CMP_limited_dir-L1cache.sm";
include "MOESI_CMP_limited_dir-L2cache.sm";
include "MOESI_CMP_limited_dir-dma.sm";
include "MOESI_CMP_limited_dir-dir.sm";
```

### 3.2 Fichier : `src/mem/ruby/protocol/MOESI_CMP_limited_dir-msg.sm`

**Action :** Copier `MOESI_CMP_directory-msg.sm` sans modification à ce stade.

**Modifications futures (Phase 2) :** Ajouter éventuellement un `CoherenceRequestType:RECALL` pour les messages de recall.

### 3.3 Fichier : `src/mem/ruby/protocol/MOESI_CMP_limited_dir-L1cache.sm`

**Action :** Copier `MOESI_CMP_directory-L1cache.sm` intégralement.

**Modifications :** Aucune pour les Phases 2 et 3. Le L1 ne change pas car :
- Les évictions explicites (PUTS, PUTX, PUTO) sont déjà en place.
- Le L1 ne connaît pas la structure interne du directory.

### 3.4 Fichier : `src/mem/ruby/protocol/MOESI_CMP_limited_dir-L2cache.sm`

**Action :** Copier `MOESI_CMP_directory-L2cache.sm` intégralement.

**Modifications :** Aucune pour la Phase 2. Potentiellement mineures pour la Phase 3 si on veut gérer le recall depuis le L2 (mais pas nécessaire au début).

### 3.5 Fichier : `src/mem/ruby/protocol/MOESI_CMP_limited_dir-dma.sm`

**Action :** Copier `MOESI_CMP_directory-dma.sm` intégralement.

**Modifications :** Aucune.

### 3.6 Fichier : `src/mem/ruby/protocol/MOESI_CMP_limited_dir-dir.sm`

**Action :** Copier `MOESI_CMP_directory-dir.sm`. Ce fichier sera le cœur des modifications dans les Phases 2 et 3.

### 3.7 Fichier : `src/mem/ruby/protocol/SConsopts`

**Action :** Vérifier si ce fichier enregistre les protocoles ou si c'est fait via Kconfig. Dans les versions récentes de gem5, c'est via Kconfig (`build_opts/`). Pour les anciennes versions :

**Ajouter dans SConsopts (si pertinent) :**
```python
main.Append(ALL_PROTOCOLS=['MOESI_CMP_limited_dir'])
```

**Ou via Kconfig :** Ajouter une entrée pour le nouveau protocole dans la configuration de build.

### 3.8 Validation Phase 1

À ce stade, le protocole doit compiler et fonctionner **identiquement** au protocole original. Compiler avec :
```bash
scons build/X86_MOESI_CMP_limited_dir/gem5.opt PROTOCOL=MOESI_CMP_limited_dir -j$(nproc)
```

---

## 4. Phase 2 — Directory à taille limitée

### 4.1 Fichier principal : `MOESI_CMP_limited_dir-dir.sm`

#### 4.1.1 Modification de la déclaration machine (ligne 41-60)

**Avant :**
```slicc
machine(MachineType:Directory, "Directory protocol")
: DirectoryMemory * directory;
  Cycles directory_latency := 6;
  Cycles to_memory_controller_latency := 1;
  // Message Queues (inchangées)
```

**Après :**
```slicc
machine(MachineType:Directory, "Limited Directory protocol")
: CacheMemory * directoryCache;          // NOUVEAU : remplace DirectoryMemory
  Cycles directory_latency := 6;
  Cycles to_memory_controller_latency := 1;
  int max_directory_entries := 1024;      // NOUVEAU : paramètre de taille
  // Message Queues (inchangées)
```

**Explication :** `CacheMemory` apporte gratuitement l'associativité, la politique de remplacement (LRU), et les méthodes `isTagPresent()`, `cacheAvail()`, `cacheProbe()` pour trouver la victime.

#### 4.1.2 Modification de la structure Entry (ligne 123-128)

**Pas de changement à ce stade.** La structure reste identique :
```slicc
structure(Entry, desc="...", interface='AbstractCacheEntry', main="false") {
    State DirectoryState;
    NetDest Sharers;
    NetDest Owner;
    int WaitingUnblocks;
}
```

**Note :** le `main="false"` devra devenir `main="true"` ou être supprimé si on utilise `CacheMemory`, car la politique de remplacement doit pouvoir gérer cette entrée. **Attention : c'est un point critique à vérifier.**

Alternativement, garder `main="false"` et gérer le remplacement manuellement (pas recommandé).

#### 4.1.3 Modification de `getDirectoryEntry` (ligne 156-160)

**Avant :**
```slicc
Entry getDirectoryEntry(Addr addr), return_by_pointer="yes" {
    Entry dir_entry := static_cast(Entry, "pointer", directory[addr]);
    assert(is_valid(dir_entry));
    return dir_entry;
}
```

**Après :**
```slicc
Entry getDirectoryEntry(Addr addr), return_by_pointer="yes" {
    Entry dir_entry := static_cast(Entry, "pointer", directoryCache[addr]);
    assert(is_valid(dir_entry));
    return dir_entry;
}
```

#### 4.1.4 Modification de `allocateDirectoryEntry` (ligne 162-166)

**Avant :**
```slicc
Entry allocateDirectoryEntry(Addr addr), return_by_pointer="yes" {
    Entry dir_entry := static_cast(Entry, "pointer",
                              directory.allocate(addr, new Entry));
    return dir_entry;
}
```

**Après :**
```slicc
Entry allocateDirectoryEntry(Addr addr), return_by_pointer="yes" {
    // Vérifier si le cache a de la place
    if (directoryCache.cacheAvail(addr) == false) {
        // Pas de place → il faut trigger un recall sur une victime
        // On ne peut PAS allouer directement ici, il faut d'abord évicter
        // → Voir le mécanisme de recall ci-dessous
        error("Should not reach here: recall should have been triggered");
    }
    Entry dir_entry := static_cast(Entry, "pointer",
                              directoryCache.allocate(addr, new Entry));
    return dir_entry;
}
```

#### 4.1.5 Modification de `deallocateDirectoryEntry` (ligne 168-176)

**Avant :**
```slicc
void deallocateDirectoryEntry(Addr addr) {
    assert(getDirectoryEntry(addr).DirectoryState != State:I);
    assert(getDirectoryEntry(addr).Owner.count() == 0);
    assert(getDirectoryEntry(addr).Sharers.count() == 0);
    directory.deallocate(addr);
}
```

**Après :**
```slicc
void deallocateDirectoryEntry(Addr addr) {
    // Les assertions restent pour vérifier la cohérence
    assert(getDirectoryEntry(addr).Owner.count() == 0);
    assert(getDirectoryEntry(addr).Sharers.count() == 0);
    directoryCache.deallocate(addr);
}
```

#### 4.1.6 Modification de `getState` (ligne 178-186)

**Avant :**
```slicc
State getState(TBE tbe, Addr addr) {
    Entry dir_entry := static_cast(Entry, "pointer", directory[addr]);
    if (is_valid(dir_entry)) {
      return dir_entry.DirectoryState;
    } else {
      return State:I;
    }
}
```

**Après :**
```slicc
State getState(TBE tbe, Addr addr) {
    Entry dir_entry := static_cast(Entry, "pointer", directoryCache[addr]);
    if (is_valid(dir_entry)) {
      return dir_entry.DirectoryState;
    } else {
      return State:I;  // Pas dans le cache → considéré comme I
    }
}
```

**Point important :** Quand l'entrée n'est pas dans le directory cache, l'état est I. Cela signifie que la mémoire est à jour et aucun cache n'a de copie (ou alors l'entrée a été recall-ée et tous les caches ont été invalidés).

#### 4.1.7 Modification de `isPresent` / ajout de fonctions (lignes 189+)

**Ajouter :**
```slicc
bool isDirectoryPresent(Addr addr) {
    return directoryCache.isTagPresent(addr);
}

bool isDirectoryCacheAvail(Addr addr) {
    return directoryCache.cacheAvail(addr);
}

Addr directoryProbeVictim(Addr addr) {
    return directoryCache.cacheProbe(addr);
}
```

#### 4.1.8 Ajout de nouveaux états transitoires (dans state_declaration, ligne 63-95)

**Ajouter après les états existants :**
```slicc
    // Recall states (Phase 2)
    S_Recall, AccessPermission:Busy, desc="Was S, recall in progress, sending INVs to all sharers";
    O_Recall, AccessPermission:Busy, desc="Was O, recall in progress, asking owner for WB + INVs";
    M_Recall, AccessPermission:Busy, desc="Was M, recall in progress, asking owner for WB";
```

#### 4.1.9 Ajout de nouveaux événements (dans enumeration Event, ligne 98-118)

**Ajouter :**
```slicc
    Recall,         desc="Directory cache is full, must evict an entry";
    Recall_Ack,     desc="Sharer acknowledged recall invalidation";
    Last_Recall_Ack, desc="Last recall ack received";
```

#### 4.1.10 Ajout de nouvelles actions pour le recall

**Ajouter les actions suivantes (après les actions existantes, ~ligne 399+) :**

```slicc
action(recall_triggerEviction, "re", desc="Check if directory cache needs eviction before allocating") {
    // Si le cache n'a pas de place pour la nouvelle adresse,
    // trouver la victime et déclencher un recall
    if (directoryCache.cacheAvail(address) == false) {
        Addr victim := directoryCache.cacheProbe(address);
        // Trigger le recall sur la victime
        trigger(Event:Recall, victim, TBEs[victim]);
    }
}

action(recall_sendInvalidationsToSharers, "ris", desc="Send recall INVs to all sharers of the victim") {
    enqueue(forwardNetwork_out, RequestMsg, directory_latency) {
        out_msg.addr := address;
        out_msg.Type := CoherenceRequestType:INV;
        out_msg.Requestor := machineID;  // Le directory lui-même demande
        out_msg.RequestorMachine := MachineType:Directory;
        out_msg.Destination.addNetDest(getDirectoryEntry(address).Sharers);
        out_msg.MessageSize := MessageSizeType:Invalidate_Control;
    }
}

action(recall_forwardWritebackToOwner, "rfw", desc="Ask owner to writeback for recall") {
    enqueue(forwardNetwork_out, RequestMsg, directory_latency) {
        out_msg.addr := address;
        out_msg.Type := CoherenceRequestType:GETX;
        out_msg.Requestor := machineID;
        out_msg.RequestorMachine := MachineType:Directory;
        out_msg.Destination.addNetDest(getDirectoryEntry(address).Owner);
        out_msg.Acks := 0;
        out_msg.MessageSize := MessageSizeType:Forwarded_Control;
    }
}
```

#### 4.1.11 Ajout de transitions pour le recall

**Ajouter :**
```slicc
// Recall depuis l'état S : invalider tous les sharers
transition(S, Recall, S_Recall) {
    v_allocateTBE;
    recall_sendInvalidationsToSharers;
    // Ne pas pop : le Recall est interne
}

// Quand tous les sharers ont répondu (recall complété)
transition(S_Recall, Last_Recall_Ack, I) {
    cc_clearSharers;
    w_deallocateTBE;
    deallocDirEntry;
    // popTriggerQueue ou mécanisme interne
}

// Recall depuis l'état M : demander writeback au owner
transition(M, Recall, M_Recall) {
    v_allocateTBE;
    recall_forwardWritebackToOwner;
}

// Le owner répond avec un dirty writeback
transition(M_Recall, Dirty_Writeback, WBI) {
    c_clearOwner;
    qw_queueMemoryWBFromCacheRequest;
    i_popIncomingRequestQueue;
}

transition(M_Recall, Clean_Writeback, I) {
    c_clearOwner;
    w_deallocateTBE;
    deallocDirEntry;
    i_popIncomingRequestQueue;
}

// Recall depuis O : WB du owner + INV des sharers
transition(O, Recall, O_Recall) {
    v_allocateTBE;
    recall_forwardWritebackToOwner;
    recall_sendInvalidationsToSharers;
}
```

#### 4.1.12 Modification des transitions `I + GETS` et `I + GETX`

**Le problème :** Actuellement, quand une requête GETS/GETX arrive et l'état est I, on alloue une entrée (`allocDirEntry`). Avec un directory limité, il faut d'abord vérifier s'il y a de la place.

**Approche 1 (simple) :** Avant d'appeler `allocDirEntry`, vérifier la disponibilité et recycler si le cache est plein (le recall sera géré séparément).

**Approche 2 (meilleure) :** Déclencher le recall comme un événement interne. Quand le recall est terminé, re-traiter la requête originale.

**Modification de la transition I + GETS :**

```slicc
// AVANT (original) :
transition(I, GETS, IS_M) {
    allocDirEntry;
    v_allocateTBE;
    qf_queueMemoryFetchRequest;
    i_popIncomingRequestQueue;
}

// APRÈS :
// Si pas de place, recycler (le recall sera traité en parallèle)
transition(I, GETS) {
    if (directoryCache.cacheAvail(address) == false) {
        // Pas de place → recycler la requête et trigger un recall
        zz_recycleRequest;
        // Le recall sera triggered par un mécanisme séparé (voir ci-dessous)
    } else {
        // Place disponible → procéder normalement
        // ... transition vers IS_M
    }
}
```

**Note SLICC :** On ne peut pas mettre de `if` directement dans les transitions SLICC de cette façon. La solution est d'utiliser un `in_port` modifié qui vérifie la disponibilité avant de trigger l'événement, ou d'ajouter un événement séparé.

**Solution recommandée avec SLICC :**

Modifier le `in_port(requestQueue_in, ...)` pour ajouter un check :

```slicc
in_port(requestQueue_in, RequestMsg, requestToDir, rank=1) {
    if (requestQueue_in.isReady(clockEdge())) {
      peek(requestQueue_in, RequestMsg) {
        // NOUVEAU : vérifier si l'entrée existe déjà OU si le cache a de la place
        if (getState(TBEs[in_msg.addr], in_msg.addr) == State:I) {
            // L'état est I → on va devoir allouer
            if (directoryCache.isTagPresent(in_msg.addr) == false &&
                directoryCache.cacheAvail(in_msg.addr) == false) {
                // Pas de place → trigger recall sur la victime
                Addr victim := directoryCache.cacheProbe(in_msg.addr);
                trigger(Event:Recall, victim, TBEs[victim]);
                // La requête originale sera recyclée
            } else {
                // Traitement normal
                if (in_msg.Type == CoherenceRequestType:GETS) {
                    trigger(Event:GETS, in_msg.addr, TBEs[in_msg.addr]);
                } // ... etc
            }
        } else {
            // L'entrée existe déjà → traitement normal
            if (in_msg.Type == CoherenceRequestType:GETS) {
                trigger(Event:GETS, in_msg.addr, TBEs[in_msg.addr]);
            } // ... etc
        }
      }
    }
}
```

#### 4.1.13 Gestion du stall pendant un recall

Les requêtes arrivant pour une adresse en cours de recall doivent être recyclées :

```slicc
transition({S_Recall, O_Recall, M_Recall}, {GETS, GETX, PUTX, PUTO, PUTO_SHARERS}) {
    zz_recycleRequest;
}
```

### 4.2 Fichier : `MOESI_CMP_limited_dir-msg.sm`

**Modification (optionnelle) :** Ajouter un type de requête `RECALL` si on veut distinguer les invalidations de recall des invalidations normales.

```slicc
// Ajouter dans CoherenceRequestType :
RECALL,    desc="Recall: directory evicting this line, invalidate your copy";
```

**Avantage :** Permet au L2 de distinguer un `INV` normal (pour un GETX concurrent) d'un `RECALL` (pour libérer de la place dans le directory). Le L2 peut alors répondre différemment (ex: envoyer un ack au directory plutôt qu'au demandeur).

**Alternative :** Réutiliser `INV` avec le `Requestor` = directory comme signal implicite.

---

## 5. Phase 3 — Sharelist compacte (Ackwise)

### 5.1 Fichier principal : `MOESI_CMP_limited_dir-dir.sm`

#### 5.1.1 Modification de la structure Entry

**Avant (Phase 2) :**
```slicc
structure(Entry, desc="...", interface='AbstractCacheEntry') {
    State DirectoryState;
    NetDest Sharers;            // O(N) bits !
    NetDest Owner;
    int WaitingUnblocks;
}
```

**Après (Phase 3) :**
```slicc
structure(Entry, desc="Limited directory entry",
          interface='AbstractCacheEntry') {
    State DirectoryState;

    // Sharelist compacte (remplace NetDest Sharers) :
    bool OverflowBit,      default="false",   desc="G bit: true = mode broadcast";
    int  SharerCount,      default="0",       desc="Total number of sharers";
    int  SharerListSize,   default="0",       desc="Number of valid entries in SharerList (0..K)";
    MachineID Sharer0,                        desc="Sharer pointer 0";
    MachineID Sharer1,                        desc="Sharer pointer 1";
    MachineID Sharer2,                        desc="Sharer pointer 2";
    MachineID Sharer3,                        desc="Sharer pointer 3";
    MachineID Sharer4,                        desc="Sharer pointer 4";
    // K = 5 pointeurs (configurable)

    // Owner (inchangé conceptuellement, mais utilise MachineID au lieu de NetDest)
    NetDest Owner;
    int WaitingUnblocks;
}
```

**Alternative plus propre avec un NetDest réduit :** Garder `NetDest Sharers` mais le limiter en logique (compter les éléments et basculer en overflow). Cela simplifie le code SLICC mais ne modélise pas le vrai coût matériel.

**Recommandation pour un premier prototype :** Garder `NetDest Sharers` + ajouter `bool OverflowBit` + `int SharerCount`. La logique Ackwise sera dans les actions, pas dans la structure. Cela évite de casser toutes les actions existantes qui utilisent `Sharers.add()`, `Sharers.remove()`, etc.

**Structure finale recommandée pour le prototype :**
```slicc
structure(Entry, desc="Limited directory entry with Ackwise-like sharelist",
          interface='AbstractCacheEntry') {
    State DirectoryState;
    NetDest Sharers;             // Utilisé UNIQUEMENT quand OverflowBit == false
    NetDest Owner;
    int WaitingUnblocks;
    
    // Champs Ackwise ajoutés :
    bool OverflowBit, default="false", desc="True = sharelist has overflowed, broadcast mode";
    int SharerCount, default="0",      desc="Total sharers count (used when OverflowBit=true)";
    int MaxSharers, default="5",       desc="K = max sharers before overflow (paramètre)";
}
```

#### 5.1.2 Ajout d'un paramètre K (nombre max de sharers)

**Dans la déclaration machine :**
```slicc
machine(MachineType:Directory, "Limited Directory protocol")
: CacheMemory * directoryCache;
  Cycles directory_latency := 6;
  Cycles to_memory_controller_latency := 1;
  int max_sharers := 5;              // NOUVEAU : K = seuil Ackwise
```

#### 5.1.3 Modification de l'action `m_addUnlockerToSharers` (ligne 583-588)

**Avant :**
```slicc
action(m_addUnlockerToSharers, "m", desc="Add the unlocker to the sharer list") {
    peek(unblockNetwork_in, ResponseMsg) {
      if (in_msg.SenderMachine == MachineType:L2Cache) {
        getDirectoryEntry(address).Sharers.add(in_msg.Sender);
      }
    }
}
```

**Après :**
```slicc
action(m_addUnlockerToSharers, "m", desc="Add the unlocker to the sharer list (Ackwise)") {
    peek(unblockNetwork_in, ResponseMsg) {
      if (in_msg.SenderMachine == MachineType:L2Cache) {
        Entry dir_entry := getDirectoryEntry(address);
        if (dir_entry.OverflowBit == false) {
            // Mode précis : vérifier si on dépasse K
            if (dir_entry.Sharers.count() < max_sharers) {
                // Place disponible : ajouter normalement
                dir_entry.Sharers.add(in_msg.Sender);
                dir_entry.SharerCount := dir_entry.SharerCount + 1;
            } else {
                // Overflow ! Passer en mode broadcast
                dir_entry.OverflowBit := true;
                dir_entry.SharerCount := dir_entry.Sharers.count() + 1;
                dir_entry.Sharers.clear();  // On n'a plus besoin de la liste
            }
        } else {
            // Mode broadcast : juste incrémenter le compteur
            dir_entry.SharerCount := dir_entry.SharerCount + 1;
        }
      }
    }
}
```

#### 5.1.4 Modification de l'action `g_sendInvalidations` (ligne 541-558)

**Avant :**
```slicc
action(g_sendInvalidations, "g", desc="Send invalidations to sharers, not including the requester") {
    peek(requestQueue_in, RequestMsg) {
      if ((getDirectoryEntry(in_msg.addr).Sharers.count() > 1) ||
          ((getDirectoryEntry(in_msg.addr).Sharers.count() > 0) &&
           (getDirectoryEntry(in_msg.addr).Sharers.isElement(in_msg.Requestor) == false))) {
        enqueue(forwardNetwork_out, RequestMsg, directory_latency) {
          out_msg.addr := address;
          out_msg.Type := CoherenceRequestType:INV;
          out_msg.Requestor := in_msg.Requestor;
          out_msg.RequestorMachine := machineIDToMachineType(in_msg.Requestor);
          out_msg.Destination.addNetDest(getDirectoryEntry(in_msg.addr).Sharers);
          out_msg.Destination.remove(in_msg.Requestor);
          out_msg.MessageSize := MessageSizeType:Invalidate_Control;
        }
      }
    }
}
```

**Après :**
```slicc
action(g_sendInvalidations, "g", desc="Send invalidations (Ackwise: multicast or broadcast)") {
    peek(requestQueue_in, RequestMsg) {
      Entry dir_entry := getDirectoryEntry(in_msg.addr);

      if (dir_entry.OverflowBit == false) {
        // MODE PRÉCIS : multicast ciblé aux sharers connus
        if ((dir_entry.Sharers.count() > 1) ||
            ((dir_entry.Sharers.count() > 0) &&
             (dir_entry.Sharers.isElement(in_msg.Requestor) == false))) {
          enqueue(forwardNetwork_out, RequestMsg, directory_latency) {
            out_msg.addr := address;
            out_msg.Type := CoherenceRequestType:INV;
            out_msg.Requestor := in_msg.Requestor;
            out_msg.RequestorMachine := machineIDToMachineType(in_msg.Requestor);
            out_msg.Destination.addNetDest(dir_entry.Sharers);
            out_msg.Destination.remove(in_msg.Requestor);
            out_msg.MessageSize := MessageSizeType:Invalidate_Control;
          }
        }
      } else {
        // MODE BROADCAST : envoyer à TOUS les nœuds L2Cache
        if (dir_entry.SharerCount > 0) {
          enqueue(forwardNetwork_out, RequestMsg, directory_latency) {
            out_msg.addr := address;
            out_msg.Type := CoherenceRequestType:INV;
            out_msg.Requestor := in_msg.Requestor;
            out_msg.RequestorMachine := machineIDToMachineType(in_msg.Requestor);
            // Broadcast : ajouter TOUS les nœuds L2 comme destination
            out_msg.Destination.broadcast(MachineType:L2Cache);
            out_msg.Destination.remove(in_msg.Requestor);
            out_msg.MessageSize := MessageSizeType:Invalidate_Control;
          }
        }
      }
    }
}
```

**Note :** La méthode `broadcast(MachineType)` sur `NetDest` n'existe peut-être pas directement. Il faudra vérifier l'API de `NetDest` dans gem5. Alternative : construire le `NetDest` manuellement en ajoutant tous les MachineIDs de type L2Cache.

#### 5.1.5 Modification de l'action `d_sendDataMsg` — calcul AckCount (ligne 460-485)

**Avant :**
```slicc
if (getDirectoryEntry(in_msg.addr).Sharers.isElement(in_msg.OriginalRequestorMachId) == true) {
    out_msg.Acks := (getDirectoryEntry(in_msg.addr).Sharers.count()) - 1;
} else {
    out_msg.Acks := getDirectoryEntry(in_msg.addr).Sharers.count();
}
```

**Après :**
```slicc
Entry dir_entry := getDirectoryEntry(in_msg.addr);
if (dir_entry.OverflowBit == false) {
    // Mode précis : AckCount = nombre de sharers dans la liste
    if (dir_entry.Sharers.isElement(in_msg.OriginalRequestorMachId) == true) {
        out_msg.Acks := (dir_entry.Sharers.count()) - 1;
    } else {
        out_msg.Acks := dir_entry.Sharers.count();
    }
} else {
    // Mode broadcast : AckCount = SharerCount total
    // Le demandeur peut être un sharer → décrémenter de 1
    out_msg.Acks := dir_entry.SharerCount - 1;
    // Note : on ne sait pas si le demandeur est un sharer, on suppose que oui
    // (il va s'auto-ignorer si l'INV lui arrive)
}
```

#### 5.1.6 Modification de `cc_clearSharers` (ligne 456-458)

**Avant :**
```slicc
action(cc_clearSharers, "\c", desc="Clear the sharers field") {
    getDirectoryEntry(address).Sharers.clear();
}
```

**Après :**
```slicc
action(cc_clearSharers, "\c", desc="Clear the sharers field (Ackwise)") {
    Entry dir_entry := getDirectoryEntry(address);
    dir_entry.Sharers.clear();
    dir_entry.OverflowBit := false;
    dir_entry.SharerCount := 0;
}
```

#### 5.1.7 Ajout d'une action `removeSharerFromDirectory`

Pour les transitions PutS (quand un sharer se retire), ajouter :

```slicc
action(ackwise_removeSharer, "ars", desc="Remove a sharer in Ackwise mode") {
    peek(requestQueue_in, RequestMsg) {
        Entry dir_entry := getDirectoryEntry(address);
        if (dir_entry.OverflowBit == false) {
            // Mode précis : retirer le sharer de la liste
            dir_entry.Sharers.remove(in_msg.Requestor);
            dir_entry.SharerCount := dir_entry.SharerCount - 1;
        } else {
            // Mode broadcast : décrémenter le compteur
            dir_entry.SharerCount := dir_entry.SharerCount - 1;
            // Note : on ne peut PAS revenir en mode précis car on a perdu les identités
            // Optionnel : si SharerCount == 0, passer en OverflowBit = false
            if (dir_entry.SharerCount == 0) {
                dir_entry.OverflowBit := false;
            }
        }
    }
}
```

#### 5.1.8 Statistiques à ajouter

Ajouter des compteurs pour mesurer l'efficacité :

```slicc
// Dans la structure de la machine ou via un mécanisme de stats gem5 :
// - Nombre de transitions G=0 → G=1 (overflows)
// - Nombre de broadcasts déclenchés (vs multicasts ciblés)
// - Nombre de recalls
// - Hit rate du directory cache
```

### 5.2 Impact sur `MOESI_CMP_limited_dir-L2cache.sm`

**Si on utilise le broadcast :** Le L2Cache recevra des INV pour des lignes qu'il ne possède pas. Il doit pouvoir les ignorer gracieusement (envoyer un ACK même s'il n'a pas la ligne).

**Vérification nécessaire :** Dans le protocole original, la transition `I + Inv` existe déjà :
```slicc
transition(I, Inv) {
    i_allocateTBE;
    t_recordFwdXID;
    e_sendAck;
    s_deallocateTBE;
    m_popRequestQueue;
}
```

Cela signifie que le L2 **gère déjà le cas** où il reçoit un INV alors qu'il n'a pas la ligne (il envoie un ACK). **Pas de modification nécessaire** pour le mode broadcast basique.

**Attention :** L'ACK envoyé par un L2 qui n'a pas la ligne est envoyé au **Requestor** (le demandeur du GETX), pas au directory. Or en mode broadcast Ackwise, le directory doit compter les ACKs. Il faudra vérifier que le mécanisme d'AckCount fonctionne correctement (les ACKs vont au demandeur, le demandeur attend `Acks` acquittements).

---

## 6. Phase 4 — Configuration Python et compilation

### 6.1 Fichier : `configs/ruby/MOESI_CMP_limited_dir.py`

**Action :** Copier `configs/ruby/MOESI_CMP_directory.py` et modifier.

**Modifications :**

1. **Ligne 73 :** Changer le nom du protocole vérifié :
```python
if buildEnv["PROTOCOL"] != "MOESI_CMP_limited_dir":
    panic("This script requires the MOESI_CMP_limited_dir protocol.")
```

2. **Lignes 114, 180, 244 :** Changer les noms des contrôleurs :
```python
l1_cntrl = MOESI_CMP_limited_dir_L1Cache_Controller(...)
l2_cntrl = MOESI_CMP_limited_dir_L2Cache_Controller(...)
dma_cntrl = MOESI_CMP_limited_dir_DMA_Controller(...)
```

3. **Section directory (après ligne 216) :** Remplacer la création du directory.

**Avant (via `create_directories` dans Ruby.py) :**
```python
# Dans Ruby.py, create_directories() fait :
dir_cntrl.directory = RubyDirectoryMemory(block_size=ruby_system.block_size_bytes)
```

**Après (dans notre script, surcharger la création) :**
```python
# Ne plus utiliser create_directories() tel quel
# Créer manuellement les directory controllers avec un CacheMemory

class DirectoryCache(RubyCache):
    """Cache utilisé comme directory à taille limitée"""
    dataAccessLatency = 6
    tagAccessLatency = 6

for i in range(options.num_dirs):
    dir_cache = DirectoryCache(
        size="64kB",         # Taille du directory cache (paramètre à explorer)
        assoc=16,            # Associativité
        start_index_bit=block_size_bits,
    )
    
    dir_cntrl = MOESI_CMP_limited_dir_Directory_Controller(
        version=i,
        directoryCache=dir_cache,     # NOUVEAU paramètre
        max_sharers=5,                 # K = 5 (Ackwise)
        transitions_per_cycle=options.ports,
        ruby_system=ruby_system,
    )
    dir_cntrl.addr_ranges = [system.mem_ranges[i]]  # ou interleaved
    
    exec("ruby_system.dir_cntrl%d = dir_cntrl" % i)
    dir_cntrl_nodes.append(dir_cntrl)
```

4. **Ajout d'options en ligne de commande :**
```python
def define_options(parser):
    parser.add_option("--dir-cache-size", type="string", default="64kB",
                      help="Size of the directory cache")
    parser.add_option("--dir-cache-assoc", type="int", default=16,
                      help="Associativity of the directory cache")
    parser.add_option("--max-sharers", type="int", default=5,
                      help="K: max sharers before overflow (Ackwise threshold)")
```

### 6.2 Fichier : Configuration de build (Kconfig ou build_opts)

**Pour gem5 >= 23.1 (Kconfig) :**

Vérifier/créer un fichier de configuration pour le nouveau protocole. Typiquement dans `src/mem/ruby/protocol/Kconfig` ou `build_opts/`.

```bash
# Commande de build :
scons defconfig build/X86_LIMITED build_opts/X86
scons setconfig build/X86_LIMITED RUBY_PROTOCOL_MOESI_CMP_LIMITED_DIR=y
scons build/X86_LIMITED/gem5.opt -j$(nproc)
```

**Pour gem5 <= 23.0 (SCons direct) :**
```bash
scons build/X86_LIMITED/gem5.opt --default=X86 PROTOCOL=MOESI_CMP_limited_dir -j$(nproc)
```

### 6.3 Fichier : `src/mem/ruby/protocol/SConsopts`

**Si nécessaire** (dépend de la version de gem5), ajouter le nouveau protocole dans le système de build. Vérifier comment les autres protocoles sont enregistrés.

---

## 7. Phase 5 — Tests et validation

### 7.1 Test de non-régression (Phase 1)

```bash
# Vérifier que le protocole copié fonctionne identiquement à l'original
./build/X86_LIMITED/gem5.opt configs/example/se.py \
    --ruby --num-cpus=4 --num-l2caches=4 \
    -c tests/test-progs/hello/bin/x86/linux/hello
```

### 7.2 Test du directory limité (Phase 2)

```bash
# Tester avec différentes tailles de directory cache
for size in 1kB 4kB 16kB 64kB 256kB; do
    ./build/X86_LIMITED/gem5.opt configs/example/se.py \
        --ruby --num-cpus=4 --num-l2caches=4 \
        --dir-cache-size=$size \
        -c benchmark
done
```

**Vérifications :**
- Le système ne deadlock pas.
- Les recalls se déclenchent quand le directory est plein.
- Les données restent cohérentes (vérifier avec `--debug-flags=RubySlicc`).

### 7.3 Test de la sharelist limitée (Phase 3)

```bash
# Tester avec différentes valeurs de K
for k in 1 2 3 4 5 8 16; do
    ./build/X86_LIMITED/gem5.opt configs/example/se.py \
        --ruby --num-cpus=16 --num-l2caches=16 \
        --max-sharers=$k \
        -c benchmark_with_sharing
done
```

**Métriques à collecter :**
- Nombre de broadcasts déclenchés
- Trafic réseau total
- Latence moyenne des accès mémoire
- IPC

### 7.4 Debug

```bash
# Traces détaillées du protocole
./build/X86_LIMITED/gem5.opt --debug-flags=RubySlicc,RubyCache,RubyNetwork \
    configs/example/se.py --ruby ...

# Tableaux HTML des transitions (compiler avec SLICC_HTML=y)
# → build/X86_LIMITED/mem/protocol/html/
```

---

## 8. Dépendances entre les modifications

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DÉPENDANCES                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Phase 1 (copie)                                                         │
│      │                                                                    │
│      ├── Ne dépend de rien                                               │
│      │                                                                    │
│  Phase 2 (dir limité)                                                    │
│      │                                                                    │
│      ├── Dépend de Phase 1                                               │
│      ├── Nécessite de comprendre CacheMemory API :                       │
│      │     - isTagPresent(addr) → bool                                   │
│      │     - cacheAvail(addr) → bool                                     │
│      │     - cacheProbe(addr) → Addr (victime LRU)                       │
│      │     - allocate(addr, entry) → AbstractCacheEntry*                  │
│      │     - deallocate(addr)                                            │
│      │     - operator[](addr) → AbstractCacheEntry* (lookup)             │
│      ├── Nécessite de comprendre le mécanisme de TBE pour gérer          │
│      │   les recalls (états transitoires multiples en parallèle)         │
│      └── Peut nécessiter une triggerQueue dédiée pour les recalls        │
│                                                                           │
│  Phase 3 (sharelist compacte)                                            │
│      │                                                                    │
│      ├── Dépend de Phase 2 (directory limité déjà fonctionnel)           │
│      ├── Nécessite de comprendre NetDest API :                           │
│      │     - add(MachineID), remove(MachineID)                           │
│      │     - count() → int                                               │
│      │     - isElement(MachineID) → bool                                 │
│      │     - clear()                                                     │
│      │     - addNetDest(NetDest), broadcast(MachineType) ← à vérifier   │
│      │     - smallestElement(MachineType) → MachineID                    │
│      └── Impact minimal sur les autres fichiers .sm                      │
│                                                                           │
│  Phase 4 (config Python)                                                 │
│      │                                                                    │
│      ├── Dépend de Phases 1+2+3 (le .sm doit compiler)                   │
│      └── Peut être développée en parallèle des .sm                       │
│                                                                           │
│  Phase 5 (tests)                                                         │
│      │                                                                    │
│      └── Dépend de tout (compilation réussie + config fonctionnelle)     │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Annexe A — Résumé des fichiers à créer/modifier

| Fichier | Action | Phase | Effort |
|---------|--------|-------|--------|
| `protocol/MOESI_CMP_limited_dir.slicc` | Créer | 1 | Trivial |
| `protocol/MOESI_CMP_limited_dir-dir.sm` | Créer + modifier fortement | 1,2,3 | **ÉLEVÉ** |
| `protocol/MOESI_CMP_limited_dir-msg.sm` | Créer + modifier léger | 1,2 | Faible |
| `protocol/MOESI_CMP_limited_dir-L1cache.sm` | Créer (copie pure) | 1 | Trivial |
| `protocol/MOESI_CMP_limited_dir-L2cache.sm` | Créer (copie pure) | 1 | Trivial |
| `protocol/MOESI_CMP_limited_dir-dma.sm` | Créer (copie pure) | 1 | Trivial |
| `protocol/SConsopts` ou Kconfig | Modifier | 4 | Faible |
| `configs/ruby/MOESI_CMP_limited_dir.py` | Créer | 4 | Moyen |
| `structures/DirectoryMemory.{hh,cc,py}` | **Ne pas modifier** (lire) | — | — |
| `structures/CacheMemory.{hh,cc}` | **Ne pas modifier** (lire) | — | — |

---

## Annexe B — Points d'attention et pièges potentiels

### B.1 Le problème `main="false"` dans l'Entry

Dans le protocole original, `Entry` a `main="false"` ce qui signifie que la politique de remplacement du cache ignore cette entrée. Si on utilise `CacheMemory` comme directory, il faut que les entrées soient gérées par la replacement policy. **Retirer `main="false"`** ou le mettre à `main="true"`.

### B.2 Deadlock potentiel avec les recalls

Si le directory est en train de recall une entrée (état transitoire S_Recall) et reçoit une requête pour cette même adresse, il faut la recycler. Mais si la requête est recyclée indéfiniment car le recall ne termine jamais (le sharer est lui-même bloqué en attente du directory), c'est un deadlock.

**Solution :** S'assurer que les recalls utilisent un réseau virtuel différent ou que les réponses aux recalls ne sont pas bloquées par les requêtes normales. Le protocole utilise déjà 3 vnets, ce qui devrait suffire.

### B.3 Le broadcast dans gem5/Garnet

Garnet ne supporte pas nativement le broadcast. Un message avec `NetDest` contenant tous les nœuds est simplement répliqué par le NetworkInterface en autant de messages unicast. C'est fonctionnellement correct mais le coût en bande passante est réel (N messages).

Pour modéliser un broadcast matériel (comme dans ATAC), il faudrait modifier Garnet. Mais pour la première implémentation, le multicast via NetDest suffit.

### B.4 Interaction Owner + Sharers en mode overflow

En mode overflow (G=1), le directory ne connaît plus les identités des sharers. Mais il connaît toujours le `Owner`. Lors d'un GETX :
1. Forward au Owner (pour les données) → inchangé
2. Broadcast INV à tous → chaque nœud qui a une copie invalide et renvoie un ACK
3. Le demandeur attend `SharerCount` ACKs → correct

Le problème : les nœuds qui n'ont PAS de copie reçoivent l'INV et doivent aussi envoyer un ACK (sinon le demandeur attend indéfiniment). **Le L2 gère déjà ce cas** (transition `I + Inv` envoie un ACK). Mais il faut vérifier que le compteur d'ACKs attendu est correct.

### B.5 Transition G=1 → G=0 (impossible dans Ackwise basique)

Dans Ackwise, une fois en mode broadcast, on ne peut PLUS revenir en mode précis (les identités sont perdues). La seule façon de revenir serait :
1. Quand `SharerCount` tombe à 0 (toutes les copies invalidées) → état I → G=0 automatiquement.
2. Ou quand un GETX exclusif est satisfait → `cc_clearSharers` remet tout à 0.

C'est déjà géré par `cc_clearSharers` modifié (section 5.1.6).

---

## Annexe C — Estimation de l'effort

| Phase | Durée estimée | Difficulté |
|-------|--------------|-----------|
| Phase 1 — Copie du protocole | 1 jour | Facile |
| Phase 2 — Directory limité | 1-2 semaines | Moyenne-difficile |
| Phase 3 — Sharelist Ackwise | 1-2 semaines | Moyenne |
| Phase 4 — Config + compilation | 2-3 jours | Facile-moyenne |
| Phase 5 — Debug + tests | 1-2 semaines | Variable (dépend des bugs) |
| **Total** | **5-8 semaines** | |

La plus grande difficulté sera le **debug** : les erreurs dans les protocoles de cohérence se manifestent par des deadlocks ou des corruptions de données silencieuses, difficiles à tracer.
