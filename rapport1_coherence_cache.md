---

# Rapport : Cohérence de Cache — *A Primer on Memory Consistency and Cache Coherence* (2e éd., Nagarajan, Sorin, Hill, Wood, 2020)

---

## 1. Fondamentaux de la cohérence de cache

### 1.1 Le problème de la cohérence

La cohérence de cache émerge dès lors que plusieurs acteurs (cœurs processeurs, contrôleurs DMA, etc.) peuvent lire et écrire dans des caches privés et une mémoire partagée. Le problème se pose de la façon suivante : si le cœur C1 modifie la valeur d'un emplacement mémoire A dans son cache local, le cœur C2 peut continuer à lire une copie périmée de A depuis son propre cache, sans jamais voir la mise à jour.

**Exemple canonique (Table 2.1 du livre) :** A vaut initialement 42 dans la mémoire et dans les caches de C1 et C2. Au temps 1, C1 écrit A = 43 dans son cache (write-back). C2 exécute en boucle `while (A == 42)` et lit indéfiniment la valeur périmée dans son cache — le système est incohérent.

### 1.2 Le modèle système de base

Le primer considère un chip multiprocesseur avec :
- Plusieurs cœurs single-thread, chacun possédant un **cache L1 privé** (physiquement adressé, write-back).
- Un **LLC (Last-Level Cache) partagé** jouant le rôle de contrôleur mémoire (côté mémoire).
- Un réseau d'interconnexion entre les cœurs et le LLC.

La cohérence de cache est maintenue à la granularité d'une **ligne de cache** (*cache block*). C'est l'unité de base du protocole.

### 1.3 L'interface de cohérence

Le protocole de cohérence expose une interface à deux méthodes vers le pipeline du cœur :
- `read-request(adresse)` → retourne une valeur
- `write-request(adresse, valeur)` → retourne un acquittement

Dans la catégorie **consistency-agnostic coherence** (la plus répandue, couverte dans ce primer), une écriture n'est retournée qu'après avoir été rendue visible à tous les autres cœurs. Le protocole de cohérence présente ainsi l'illusion d'une mémoire atomique sans caches.

### 1.4 Les invariants de cohérence (SWMR + valeur)

La cohérence est définie par **deux invariants fondamentaux** :

**Invariant SWMR (Single-Writer Multiple-Reader) :**
> Pour tout emplacement mémoire A, à tout instant logique, soit un unique cœur peut écrire (et lire) A, soit un ensemble quelconque de cœurs peut uniquement lire A.

En pratique, la durée de vie d'un bloc mémoire est découpée en **époques** : soit une époque read-write (un seul cœur), soit une époque read-only (zéro ou plusieurs cœurs).

**Invariant de valeur :**
> La valeur d'un bloc au début d'une époque est la même que la valeur à la fin de sa dernière époque read-write.

Ces deux invariants définissent formellement ce qu'est la cohérence. Les protocoles dits *d'invalidation* sont conçus explicitement pour les maintenir :
- Pour **lire** un bloc : envoyer des messages pour obtenir la valeur courante et s'assurer qu'aucun cœur n'a de copie en read-write.
- Pour **écrire** un bloc : envoyer des messages pour invalider toutes les copies read-only et read-write des autres cœurs.

### 1.5 États stables des lignes de cache : le modèle MOESI

La grande majorité des protocoles utilisent un sous-ensemble des **cinq états stables MOESI**, introduits par Sweazey et Smith. Chaque état encode une combinaison de quatre propriétés : **validité**, **sale** (*dirty*), **exclusivité** et **propriété** (*ownership*).

| État | Valide | Dirty | Exclusif | Propriétaire | Description |
|------|--------|-------|----------|--------------|-------------|
| **M** (Modified) | Oui | Oui | Oui | Oui | Seule copie valide du système, doit répondre aux requêtes, mémoire potentiellement périmée. |
| **O** (Owned) | Oui | Oui | Non | Oui | Copie read-only mais dirty, doit répondre aux requêtes, d'autres caches peuvent avoir des copies read-only (non-propriétaires). |
| **E** (Exclusive) | Oui | Non | Oui | Oui | Seule copie valide, propre (*clean*), mémoire à jour. Peut passer silencieusement à M. |
| **S** (Shared) | Oui | Non | Non | Non | Copie read-only propre, d'autres caches peuvent partager. |
| **I** (Invalid) | Non | — | — | — | Copie absente ou périmée, inutilisable. |

Le diagramme de Venn (Figure 6.4 du livre) montre que M, O, E sont des états de propriété. M et E sont exclusifs. M et O sont potentiellement sales.

**États transitoires :** En pratique, entre deux états stables, un bloc passe par des **états transitoires** (aussi nombreux que quelques dizaines dans les protocoles avancés). La notation utilisée est `XYZ` : la ligne passe de l'état stable X vers l'état stable Y, et la transition sera complète à la réception d'un événement de type Z (souvent `D` pour Data, `A` pour Ack). Exemple : `IMD` = était en I, va vers M, attend la donnée (`D`). Ces états sont maintenus dans les **MSHRs** (Miss Status Handling Registers).

**États au niveau LLC/mémoire (notation cache-centrique) :** L'état de la mémoire est défini par l'agrégat des états des caches. Si un bloc est en M dans un cache, la mémoire est en état M (elle n'a pas la copie à jour). Si aucun cache ne possède le bloc, la mémoire est en I.

---

## 2. Protocoles Snoopy (à écoute de bus)

### 2.1 Principe du snooping

Les protocoles snoopy reposent sur une idée simple : **tous les contrôleurs de cohérence observent (*snoop*) les requêtes de cohérence dans le même ordre** et réagissent collectivement de façon correcte.

Les requêtes de cohérence sont **diffusées** (*broadcast*) sur un réseau ordonné (typiquement un bus). L'**ordre total** des requêtes est garanti : chaque contrôleur observe la même séquence. Cet ordre total est le **point de sérialisation** du protocole. Dès qu'une requête est sérialisée sur le bus, la transaction est logiquement ordonnée, même si la réponse n'est pas encore arrivée.

**Point clé :** Dans un protocole snoopy, l'acquittement implicite est fourni par le fait que chaque contrôleur voit la requête sur le bus dans le même ordre. Il n'y a pas besoin de messages Ack explicites pour chaque cœur.

### 2.2 Protocole MSI snoopy (baseline)

Le protocole **MSI** est le plus fondamental. Un bloc peut être en état M, S, ou I dans un cache.

**Transactions et messages de base :**
- **GetS** : demande de permission read-only. Émis par un cache en I pour aller en S.
- **GetM** : demande de permission read-write. Émis par un cache en I ou S pour aller en M.
- **PutM** : éviction d'un bloc en M, avec la donnée.
- **Data** : réponse portant le contenu du bloc.

**Transitions stables (cache) :**
- I + Load miss → émission GetS → transition ISD → sur réception Data → S
- I + Store miss → émission GetM → transition IMD → sur réception Data → M
- S + Store miss → émission GetM → transition SMD → sur réception Data → M
- M + remplacement → émission PutM (avec donnée) → I
- S + remplacement → éviction silencieuse → I (car la mémoire reste propriétaire)
- M + Other-GetS snoopé → envoie Data au demandeur et à la mémoire → S
- M + Other-GetM snoopé → envoie Data au demandeur → I

**États transitoires du protocole non-atomique (Figure 7.8, Tables 7.8–7.9) :**
La version la plus réaliste autorise des requêtes non-atomiques (une requête peut être émise avant d'être sérialisée). Cela introduit une *fenêtre de vulnérabilité* pendant laquelle d'autres requêtes peuvent être intercalées. Les états transitoires sont alors :

- **ISAD** : en I, requête GetS émise, attend que sa propre requête soit sérialisée (bus) et les données.
- **ISD** : en I, GetS sérialisé, attend les données.
- **IMAD** : en I, requête GetM émise, attend sérialisation et données.
- **IMD** : en I, GetM sérialisé, attend données.
- **SMAD** : en S, requête GetM émise, attend sérialisation et données.
- **SMD** : en S, GetM sérialisé, attend données.
- **MIA** : en M, PutM émis, attend sa propre sérialisation.
- **IIA** : état après avoir répondu à une requête d'un autre nœud alors qu'on attendait son propre PutM.

**Gestion de la mémoire :**
La mémoire maintient un état `IorS` (elle est propriétaire et répond aux requêtes) et `M` (un cache est propriétaire, la mémoire n'a pas de copie à jour). L'état `IorSD` est l'état transitoire où la mémoire attend la donnée après un GetS quand un cache était en M.

### 2.3 Ajout de l'état Exclusive (MESI snoopy)

L'état **E** optimise le scénario commun : un cœur lit puis écrit un bloc. Dans un protocole MSI, deux transactions sont nécessaires (GetS puis GetM). Avec E, si lors du GetS personne d'autre ne partage le bloc, le cœur obtient le bloc en E et peut **silencieusement** passer à M sans émettre de GetM.

**Comment détecter qu'on est le seul lecteur ?** Deux approches :
1. Signal wired-OR sur le bus : les caches partageant le bloc assertent un signal partagé lors du GetS.
2. Maintien d'un état conservatif S/I à la mémoire : si la mémoire est en I (aucun partageur connu), elle répond avec des données marquées "Exclusif".

Le primer choisit la deuxième approche (état S conservatif à la mémoire). La mémoire distingue alors I (aucun partageur connu) et EorM (un cache a reçu le bloc en E ou M).

**Impact sur le protocole :** Le cache en E peut passer silencieusement à M. L'éviction d'un bloc en E utilise un PutM (comme M). L'état **EIA** est l'équivalent de MIA pour les blocs en E.

### 2.4 Ajout de l'état Owned (MOSI snoopy)

L'état **O** optimise la situation où un cache en M reçoit un GetS d'un autre cœur. Dans MSI, le propriétaire doit downgrader en S et copier la donnée vers la mémoire. Avec O, il suffit de downgrader en O (garder la propriété) et envoyer la donnée uniquement au demandeur.

**Deux bénéfices :**
1. Suppression du message vers la mémoire lors du passage M→O.
2. Évitement de l'écriture en mémoire si le bloc est réécrit avant son éviction.

Le propriétaire en O reste responsable de répondre aux futures requêtes. Il ne peut pas être évincé silencieusement : il doit émettre un PutM (ou PutO) avec la donnée.

### 2.5 Bus non-atomique et split-transaction (Tables 7.17–7.21)

Un bus split-transaction améliore la bande passante en permettant l'émission de nouvelles requêtes avant que la réponse à la requête précédente n'arrive. Cela supprime la propriété *Atomic Transactions*. Conséquences :

- **Nombreux nouveaux états transitoires** : un contrôleur peut recevoir des requêtes d'autres cœurs pendant qu'il attend sa propre réponse.
- **Risque de deadlock** si on bloque le traitement des requêtes entrantes : on utilise soit du **stalling** (simple mais bloquant la file FIFO), soit une version **non-stalling** avec encore plus d'états transitoires (ex. `ISDI`, `IMDS`, `IMDSI`...).
- **Réponse avant la requête** (Table 7.19) : un cœur peut recevoir la donnée pour sa requête Y avant d'avoir vu sa propre requête Y sérialisée sur le bus, ce qui nécessite des états `ISA`, `IMA`, `SMA`.

**Protocole non-stalling :** Pour éviter que les files FIFO ne bloquent le traitement séquentiel des requêtes, on traite immédiatement chaque message reçu en entrant dans de nouveaux états composites (ex. `IMDS` = en I, GetM sérialisé, attend Data, et a déjà reçu un Other-GetS). Lors de la réception de la donnée, le cœur doit impérativement effectuer au moins une opération de load/store (pour éviter le livelock).

### 2.6 Extensions du réseau (Section 7.6)

Les **réponses** (messages portant la donnée) ne nécessitent pas un ordre total ni un broadcast. Elles peuvent voyager sur un réseau séparé (crossbar, mesh, etc.) qui est plus efficace en bande passante et latence.

Les **requêtes** nécessitent un broadcast totalement ordonné, mais cela ne nécessite pas forcément un bus physique :
- **Topologie arborescente** (Sun Starfire E10000) : les requêtes sont unicastées vers la racine (point de sérialisation) puis rediffusées vers le bas.
- **Bus logique avec ordre temporel** (Timestamp Snooping) : les requêtes sont étiquetées avec un temps logique et traitées dans cet ordre.

### 2.7 Avantages et limites du snooping

**Avantages :**
- Transactions en 2 messages (requête + réponse) → faible latence.
- Conceptuellement simple.
- Pas d'état global à maintenir par cœur.

**Limites :**
- Le broadcast ne passe pas à l'échelle (*scalabilité*) : pour N cœurs, chaque requête génère N messages.
- La bande passante du réseau ordonné est un goulot d'étranglement.
- La bande passante de traitement des snoops dans chaque cache controller est limitante.

**Conclusion :** Le snooping est aujourd'hui peu utilisé dans les systèmes à grande échelle. Même pour les petits systèmes, le coût d'un réseau broadcast totalement ordonné est trop élevé comparé aux liens point-à-point suffisants pour les protocoles à répertoire.

---

## 3. Protocoles à Répertoire (Directory)

### 3.1 Principe du répertoire

Les protocoles à répertoire introduisent un **niveau d'indirection** pour éviter le broadcast. L'innovation clé est le **répertoire** (*directory*) : une structure de données qui maintient une vue globale de l'état de cohérence de chaque bloc.

**Fonctionnement de base :**
1. Un cache émet une requête **unicast** vers le répertoire (le contrôleur associé au bloc).
2. Le répertoire consulte son état et décide de l'action à prendre : soit répondre directement avec la donnée, soit **transférer** (*forward*) la requête vers le cœur qui possède la copie courante.
3. Le cœur qui reçoit la requête transférée répond **unicast** au demandeur.

**Comparaison avec le snooping :**
- Snooping : 2 étapes (requête broadcast + réponse unicast)
- Directory : 2 étapes si la mémoire est propriétaire (requête unicast + réponse unicast), ou 3 étapes si un cache est propriétaire (requête → forward → réponse), voire 4 étapes avec un message de completion.

**Point de sérialisation :** Dans un protocole à répertoire, les transactions sont ordonnées au niveau du répertoire. L'ordre entre requêtes concurrentes est déterminé par l'ordre d'arrivée au répertoire.

### 3.2 Structure du répertoire

#### 3.2.1 Entrée de répertoire complète (Full Bit-Vector)

Pour chaque bloc mémoire, le répertoire maintient une entrée comprenant :

```
| État (2 bits) | Propriétaire (log₂N bits) | Liste des partageurs (N bits) |
```

Pour N cœurs, la liste des partageurs est un **vecteur de bits one-hot** de N bits : le bit i est à 1 si le cœur i a une copie du bloc en état S.

**Coût :** Pour un système à N cœurs et M blocs mémoire, le répertoire complet nécessite O(N × M) bits de stockage. Ce coût évolue de façon quadratique avec la taille du système, ce qui pose un problème de scalabilité.

**Illustration (Figure 8.2) :** Une entrée de répertoire pour un système à N nœuds contient 2 bits d'état, log₂N bits pour l'identité du propriétaire, et N bits pour la liste des partageurs.

#### 3.2.2 Répertoire grossier (Coarse Directory)

**Principe :** Au lieu d'un bit par cache, chaque bit représente un **groupe** de K caches. Un GetM provoque l'envoi d'invalidations à **tous les K caches** du groupe, même si seuls certains partagent réellement le bloc.

**Avantage :** Réduction de la taille du répertoire d'un facteur K.

**Inconvénient :** Envoi d'invalidations inutiles (bandwidth gaspillé) et surcharge des contrôleurs de cache qui reçoivent des invalidations non pertinentes.

#### 3.2.3 Répertoire à pointeurs limités (Limited Pointer Directory, DiriX)

**Principe :** Au lieu d'un vecteur de bits de N entrées, le répertoire maintient **i pointeurs** (i ≪ N), chaque pointeur étant l'identité (log₂N bits) d'un partageur. Le coût est i × log₂N bits au lieu de N bits.

**Les études montrent** que la grande majorité des blocs ont 0 ou 1 partageur. Les pointeurs limités exploitent cette observation.

**Problème :** Que faire lorsqu'un i+1ième partageur tente de rejoindre ? Trois options (notation DiriX) :

1. **DiriB (Broadcast)** : Si les i pointeurs sont tous remplis, le répertoire passe dans un état "trop de partageurs" (*too many sharers*). Un GetM ultérieur déclenche un broadcast vers **tous** les N caches. Un cas extrême est **Dir0B** (aucun pointeur) : toutes les opérations déclenchent un broadcast. La variante originale Dir0B conserve 2 bits d'état par bloc (avec un état "Single Sharer" pour éviter le broadcast quand un seul cache partage). AMD Coherent HyperTransport implémente une variante de Dir0B sans aucun état de répertoire.

2. **DiriNB (No Broadcast)** : Si les i pointeurs sont pleins, le répertoire demande à l'un des partageurs actuels de s'invalider pour faire de la place. Problématique pour les blocs très largement partagés (caches d'instructions).

3. **DiriSW (Software)** : Si les i pointeurs sont pleins, le système déclenche un trap logiciel qui peut maintenir une liste complète dans des structures logicielles. Grande flexibilité mais coût de performance élevé.

#### 3.2.4 Organisation du répertoire en mémoire cache

**Répertoire en DRAM (Section 8.6.1) :** L'approche classique des multiprocesseurs multi-puces : le répertoire complet est stocké en DRAM, avec un cache de répertoire on-chip pour réduire la latence. Inconvénients : coût élevé en DRAM, risque de miss dans le cache de répertoire même pour des données présentes dans le LLC.

**Cache de répertoire inclusif embarqué dans le LLC inclusif (Section 8.6.2) :** Si le LLC est inclusif (contient tous les blocs des caches L1), il suffit d'**ajouter des bits d'état** de cohérence à chaque entrée du LLC. Un miss dans le LLC implique que le bloc est en état I dans tous les L1. Inconvénient : le LLC doit être highly-associative et gérer des *Recall requests* lors des évictions.

**Cache de répertoire inclusif standalone (Section 8.6.2) :** Structure séparée du LLC, contenant des copies des tags de tous les L1 caches. Doit être C×K-way associatif (C cœurs, K-way L1). Nécessite des PutS explicites pour maintenir la cohérence du répertoire.

**Null Directory Cache (Section 8.6.3) :** Pas de répertoire du tout. Chaque requête est broadcastée à tous les caches (Dir0B). Simple pour les petits systèmes, aucun coût de stockage.

### 3.3 Protocole MSI à répertoire (baseline)

#### 3.3.1 Modèle système

Réseau d'interconnexion quelconque (mesh, tore, etc.) qui garantit l'**ordre point-à-point** (*point-to-point ordering*) : si A envoie deux messages à B, ils arrivent dans l'ordre d'envoi. Cette propriété simplifie considérablement la gestion des races.

#### 3.3.2 Spécification de haut niveau (Figure 8.3)

**Trois classes de messages :**
- **Requêtes** (Request network) : GetS, GetM, PutS, PutM+data
- **Requêtes transférées** (Forwarded Request network, avec ordre point-à-point) : Fwd-GetS, Fwd-GetM, Inv(alidation), Put-Ack
- **Réponses** (Response network) : Data, Inv-Ack

**Transactions principales :**

*I → S (le répertoire est propriétaire) :*
1. Cache envoie GetS → répertoire
2. Répertoire envoie Data au demandeur, l'ajoute aux partageurs

*I → S (un cache est propriétaire en M) :*
1. Cache envoie GetS → répertoire
2. Répertoire envoie Fwd-GetS → propriétaire (état Dir : SD)
3. Propriétaire envoie Data → demandeur ET Data → répertoire
4. Répertoire copie la donnée en mémoire, passe en S

*I ou S → M (répertoire en I) :*
1. Cache envoie GetM → répertoire
2. Répertoire envoie Data (avec AckCount=0) au demandeur, passe en M

*I ou S → M (répertoire en S, avec partageurs) :*
1. Cache envoie GetM → répertoire
2. Répertoire envoie Data + AckCount (= nombre de partageurs) au demandeur
3. Répertoire envoie Inv(alidation) à chaque partageur
4. Chaque partageur invalide son bloc, envoie Inv-Ack → demandeur
5. Quand le demandeur reçoit toutes les Inv-Ack, il passe en M

*I ou S → M (répertoire en M, cache propriétaire) :*
1. Cache envoie GetM → répertoire
2. Répertoire envoie Fwd-GetM → propriétaire actuel
3. Propriétaire envoie Data (AckCount=0) → demandeur, passe en I

*M → I (éviction) :*
1. Cache envoie PutM+data → répertoire
2. Répertoire copie la donnée, envoie Put-Ack, passe en I

*S → I (éviction) :*
1. Cache envoie PutS → répertoire
2. Répertoire retire le cœur de la liste des partageurs, envoie Put-Ack

#### 3.3.3 États transitoires (Table 8.1)

Les états transitoires du cache sont de la forme XY^AD (X=état source, Y=état destination, A=attend Acks, D=attend Data) :

| État | Signification |
|------|---------------|
| **ISD** | I→S, attend Data |
| **IMAD** | I→M, attend Data et Ack(s) |
| **IMA** | I→M, Data reçue, attend Ack(s) restants |
| **SMAD** | S→M, attend Data et Ack(s) |
| **SMA** | S→M, Data reçue, attend Ack(s) restants |
| **MIA** | M→I (PutM envoyé), attend Put-Ack |
| **SIA** | S→I (PutS envoyé), attend Put-Ack ; effectivement en S jusqu'à réception |
| **IIA** | effectivement en I, attend Put-Ack |

**États transitoires du répertoire :**
| État | Signification |
|------|---------------|
| **SD** | Répertoire a forwardé GetS à un propriétaire M, attend Data |

**Nécessité des états transitoires :** Les transactions ne sont pas atomiques. Entre l'émission d'une requête par un cache et la réception de tous les messages nécessaires (Data + Inv-Acks), d'autres événements peuvent survenir (Fwd-GetS, Invalidation d'un autre cœur...). Les états transitoires encodent exactement quels messages manquent encore et comment réagir aux événements intercalés.

**Exemple de race (IMA + Fwd-GetS) :** Un cache est en IMA (a reçu la donnée, attend encore des Inv-Acks). Le répertoire a déjà changé son état en M (propriétaire = ce cache). Si un Fwd-GetS arrive en IMA, le cache doit répondre à la GetS, mais il n'a pas encore tous ses Inv-Acks. Le protocole le fait attendre (*stall*) et traite le Fwd-GetS quand le dernier Inv-Ack arrive.

#### 3.3.4 Mécanisme d'acquittement (Ack)

L'acquittement est le mécanisme central qui garantit que le demandeur d'un GetM sait quand tous les partageurs ont invalidé leur copie.

**Protocole :** Le répertoire inclut dans sa réponse Data un **AckCount** (nombre de partageurs à invalider). Chaque partageur, à réception d'une Inv(alidation), invalide son bloc et envoie un **Inv-Ack** directement au demandeur (pas au répertoire). Le demandeur maintient un compteur et passe en M quand il atteint zéro (état **Last-Inv-Ack** dans les tables).

**Subtilité :** La Data et les Inv-Ack peuvent arriver dans n'importe quel ordre (ils voyagent sur des réseaux différents). L'état IMA (Data reçue, attend Acks) versus IMAd (Acks partiels reçus, attend Data) permettent de gérer ces cas.

### 3.4 Protocoles MESI et MOSI à répertoire

#### 3.4.1 Ajout de l'état E (MESI directory, Section 8.3)

Si lors d'un GetS le répertoire est en état I (aucun partageur), le répertoire peut répondre avec des données **Exclusif** (Figure 8.6). Le cache passe en E. La transition E→M est silencieuse.

**Particularité :** L'état E étant considéré comme un état de propriété, son éviction nécessite un **PutE** (sans données, car le bloc est propre). Le répertoire doit distinguer les PutE "last" vs "non-last" (analogue aux PutS).

**État transitoire EIA :** Cache en E qui a émis un PutE, attend le Put-Ack.

**Comportement lors d'un Fwd-GetS vers un cache en E :** Identique au cas M. Le cache répond avec Data et passe en S (ou O dans MOSI).

#### 3.4.2 Ajout de l'état O (MOSI directory, Section 8.4)

Lorsqu'un cache en M reçoit un Fwd-GetS, au lieu de downgrader en S et copier les données vers la mémoire, il passe en **O** (garde la propriété). Le répertoire passe dans un état O agrégé.

**Cas plus complexe :** Si un GetM arrive quand le répertoire est en O (un cache propriétaire + des partageurs en S), le répertoire doit :
1. Forwarder le GetM au propriétaire (en O).
2. Envoyer des Inv aux partageurs.
3. Envoyer un AckCount au demandeur.

L'état **OMAC** (O→M, attend Acks et AckCount) est nécessaire.

**PutO :** L'éviction d'un bloc en O nécessite un PutO+data (car dirty, comme M).

**Race spécifique OMAC :** Si un cache est en OMAC (a émis un GetM pour passer de O à M) et reçoit un Fwd-GetM ou Invalidation d'un autre cœur dont le GetM a été sérialisé **avant** le sien au répertoire, il doit changer son état en IMAD (il perd sa propriété et doit réattendre Data).

### 3.5 Évitement du deadlock (Section 8.2.3)

Dans un protocole à répertoire, la réception d'un message peut déclencher l'émission d'un autre message. Des dépendances circulaires peuvent créer des deadlocks.

**Solution standard : réseaux virtuels séparés**

Le protocole utilise **3 classes de messages** sur **3 réseaux distincts** (physiques ou virtuels) :

1. **Réseau de Requêtes** : GetS, GetM, PutS, PutM+data
2. **Réseau de Requêtes Transférées** (nécessite ordre point-à-point) : Fwd-GetS, Fwd-GetM, Inv, Put-Ack
3. **Réseau de Réponses** : Data, Inv-Ack

**Invariants d'acyclicité :**
- Une Requête peut causer une Requête Transférée.
- Une Requête Transférée peut causer une Réponse.
- La Réponse doit toujours être "sinkable" (ne peut pas être bloquée).
- Un contrôleur ne peut pas bloquer le traitement d'un message de classe A en attendant un message de classe B si B peut être causé par A.

**Illustration du deadlock (Figure 8.4) :** Si les requêtes et les réponses partagent le même réseau FIFO, une réponse peut être bloquée derrière une requête qui attend elle-même une réponse → deadlock. La séparation des réseaux (Figure 8.5) résout ce problème.

**Réseau virtuel (sidebar Section 9.3.1) :** Au lieu de dupliquer les liens physiques, on peut ajouter des files FIFO séparées pour chaque classe de message à chaque switch et point de terminaison. Le coût est le stockage des files supplémentaires, pas des liens.

### 3.6 Gestion des conflits et races

**Races dans ISD :** Un cache en ISD (attend Data pour un GetS) peut recevoir une Inv(alidation) avant la Data. C'est possible car Data et Inv voyagent sur des réseaux différents (pas d'ordre garanti entre eux). Le cache invalide son bloc à réception de l'Inv, puis quand Data arrive, il est en ISDI et passe en I (sans mémoriser la donnée).

**Races MIA + Fwd-GetS/M :** Un cache en MIA (a envoyé PutM, attend Put-Ack) peut recevoir un Fwd-GetS ou Fwd-GetM. Il doit répondre comme s'il était encore en M, puis passer en SIA ou IIA selon le type de forward. La Put-Ack est attendue pour finaliser.

**Protocole non-stalling (Section 8.7.2, Tables 8.8–8.9) :** Pour améliorer les performances, on peut traiter les messages transférés même en état transitoire, au lieu de bloquer. Cela introduit de nouveaux états comme **IMAS** (en IMA, a reçu un Fwd-GetS, va passer en S après avoir obtenu tous les Acks), **IMAI** (va passer en I), **IMASI** (va passer en I via S). Le nombre d'états explose, mais les performances s'améliorent.

### 3.7 Réseau sans ordre point-à-point (Section 8.7.3)

Sans garantie d'ordre point-à-point sur le réseau de requêtes transférées, des races supplémentaires apparaissent. **Exemple :** Le répertoire forwarde Fwd-GetS puis Fwd-GetM au même propriétaire. Sans ordre point-à-point, Fwd-GetM peut arriver avant Fwd-GetS. Le propriétaire répond à Fwd-GetM et passe en I. Fwd-GetS arrive, mais le cache est en I et ne peut pas répondre → deadlock.

**Solutions :** Handshaking supplémentaire (le répertoire attend un Ack de réception avant de forwarder une nouvelle requête). Cela est nécessaire si on veut utiliser du **routage adaptatif** (les messages prennent des chemins différents, ce qui peut inverser leur ordre d'arrivée).

---

## 4. Optimisations et Variantes Avancées

### 4.1 Réduction de la taille du répertoire

**Full bit-vector :** O(N) bits par bloc. Pour 1024 cœurs, c'est 128 octets par entrée de répertoire, soit 12.5% d'overhead pour des blocs de 64 octets. Inacceptable.

**Coarse directory :** Chaque bit représente K caches. Pour K=4 : facteur 4 de réduction. Mais InvalidAcK inutiles multiplié par K dans le pire cas.

**Limited pointer DiriNB :** Pour i=2 pointeurs et N=64 cœurs : 2×6=12 bits vs 64 bits. Réduction de 5×. Suffit pour la grande majorité des cas (distribution de partage très concentrée).

**Approche SGI Origin :** Choix dynamique par entrée entre représentation coarse bit-vector et représentation limited pointer selon le niveau de partage observé pour le bloc.

**HT Assist (AMD Opteron) :** Deux états seulement pour les partageurs : "un partageur" (et son ID) vs "plus d'un partageur" (broadcast). Cache de répertoire partagé avec le LLC (1 MB alloué statiquement), entrées de 4 octets organisées en 4-way set-associatif.

### 4.2 Directory cache et recalls

**Problem :** Si le cache de répertoire est limité en taille, il peut saturer. Quand une nouvelle entrée doit être ajoutée et que le set est plein, le répertoire doit **évoquer** (*recall*) un des blocs présents : il envoie des *Recall requests* à tous les caches qui le partagent, attend leurs Acks, puis libère l'entrée.

**Règle empirique (Conway et al.)** : Le cache de répertoire doit couvrir au moins la taille agrégée des caches qu'il supervise (mais peut être plus grand pour réduire les recalls).

**PutS explicite vs silencieux :** Les PutS explicites permettent de maintenir la liste des partageurs plus précisément, réduisant le nombre de recalls inutiles. Mais ils consomment de la bande passante. AMD (HT Assist) a choisi de ne pas envoyer de PutS, préférant accepter plus de recalls.

### 4.3 Mécanismes d'acquittement avancés

**GetM depuis S (optimisation MOSI) :** Dans MSI directory, si un cache est en S et émet un GetM, le répertoire lui renvoie les données + l'AckCount. Mais le cache a déjà les données (il était en S). On peut optimiser avec un message **AckCount-seulement** (sans données), d'où l'état **OMAC** (already has data, just waiting for Acks + AckCount).

**Last-Inv-Ack :** Détection du dernier acquittement pour simplifier la logique du protocole : on peut déclencher la transition finale sur l'événement `Last-Inv-Ack` plutôt que de vérifier un compteur à chaque Inv-Ack.

**AckCount piggybacking :** Dans le protocole MOSI, quand le répertoire est en O, il doit forwarder le GetM au propriétaire O et envoyer un AckCount au demandeur. L'AckCount représente le nombre de partageurs (en S) qui vont envoyer des Inv-Ack. Le propriétaire O répond avec Data+AckCount consolidé.

### 4.4 Évictions silencieuses vs explicites (Section 8.7.4)

**PutS silencieux :** Le cache évince la ligne S sans notifier le répertoire. Simple, économise la bande passante. Mais le répertoire maintient une liste de partageurs potentiellement incorrecte (sur-approximation). Peut générer des Inv inutiles lors d'un GetM.

**PutS explicite (choix du baseline) :**
- Avantage 1 : liste de partageurs précise → moins d'Inv inutiles lors des GetM.
- Avantage 2 : permet de détecter le "last sharer" → optimisation E dans MESI.
- Avantage 3 : simplifie le protocole (élimine certaines races).
- Inconvénient : messages supplémentaires GetS/PutS/Put-Ack.

### 4.5 Optimisation migratory sharing (Section 9.2.1)

Pour les patterns de partage migratoire (un cœur lit puis écrit, puis un autre cœur fait de même), on peut prévoir un état **MM (Migratory Modified)** : si un cache en M reçoit un GetS d'un autre cœur, au lieu de partager en S, il envoie une copie exclusive (invalidant sa propre copie). Si le pattern migratoire se confirme, cela économise un GetM ultérieur.

### 4.6 False sharing et sub-block coherence (Section 9.2.2)

Le false sharing survient quand deux cœurs accèdent à des données différentes sur le même bloc de cache. Solution : **sous-blocs de cohérence** (*sub-block coherence*) avec des bits d'état par sous-bloc, permettant des permissions différentes sur différentes parties d'une ligne.

### 4.7 Token coherence (Section 9.4)

Token Coherence (Martin, Hill, Wood, 2003) est une troisième classe de protocoles qui généralise snooping et directory. Chaque bloc a un nombre fixe de **tokens**. Un cœur avec ≥1 token peut lire ; un cœur avec **tous** les tokens peut écrire. Les états MSI sont équivalents à avoir tous/certains/aucun token.

Le protocole se divise en :
- **Correctness substrate** : garantit la conservation des tokens et la vivacité.
- **Performance protocol** : stratégie d'envoi des requêtes (broadcast, multicast vers les partageurs prédits, etc.).

---

## 5. Protocoles à Répertoire Commerciaux : Études de Cas

### 5.1 SGI Origin 2000 (Section 8.8.1)

- Système à jusqu'à 512 nœuds (1024 cœurs MIPS R10000).
- Protocole directory avec état E **non-propriétaire** (éviction silencieuse de E autorisée).
- Réseau **sans ordre point-à-point** → nécessite plus de handshaking pour gérer les races.
- Représentation **dynamique** du répertoire : coarse bit-vector ou limited pointer selon le niveau de partage.
- Upgrade (S→M sans donner les données) : risque de NACK si perte de la copie S pendant la fenêtre de vulnérabilité.
- Seulement **2 réseaux** (requête + réponse) au lieu de 3 : utilise des messages "backoff" pour éviter le deadlock.

### 5.2 AMD Coherent HyperTransport → HyperTransport Assist

**Coherent HT :** Dir0B sans état de répertoire. Toutes les requêtes sont broadcastées. Simple mais peu scalable et haute consommation de bande passante.

**HT Assist :** Évolution avec un cache de répertoire inclusif embarqué dans le LLC. Deux états pour les partageurs ("un" vs "plusieurs"). Pas de PutS explicites. Cache de répertoire partagé avec le LLC (1 MB statiquement alloué).

### 5.3 Intel QPI (Section 8.8.4)

Cinq états stables : MESI + **F (Forward)** — état read-only propre, un seul détenteur, peut forwarder les données (similaire à O mais propre).

**Home Snoop mode :** Protocole directory scalable standard. Requête unicast → répertoire → forward → réponse → notification au répertoire.

**Source Snoop mode :** Le demandeur broadcaste sa requête à tous les nœuds. Plus rapide (moins de latence sur le chemin critique), mais consomme plus de bande passante. Le répertoire ordonne les requêtes non pas à leur arrivée, mais à la réception des *snoop responses* de tous les nœuds.

---

## 6. Vivacité : Deadlock, Livelock, Starvation (Chapitre 9)

### 6.1 Deadlock (Section 9.3.1)

**Deadlock de protocole :** Race conditions non gérées (ex. : un cœur attend un Inv-Ack qui ne viendra jamais car le partageur a déjà envoyé un PutS).

**Deadlock de ressources cache :** Un cache alloue tous ses buffers (TBEs) pour ses propres requêtes et ne peut plus traiter les requêtes entrantes d'autres cœurs.

**Deadlock réseau dépendant du protocole :** Résolu par les 3 réseaux virtuels séparés + règles d'acyclicité :
1. Chaque classe de message sur son réseau.
2. Ordre de dépendance Requête → Fwd-Requête → Réponse.
3. Les Réponses sont toujours consommées immédiatement.

### 6.2 Livelock (Section 9.3.2)

**Fenêtre de vulnérabilité :** Un cœur en ISD observe un Other-GetM, passe en ISDI, reçoit la donnée mais doit invalider (→ I). Il réémet un GetS, mais le même scénario se répète → livelock.

**Solution :** Obliger le cœur à effectuer au moins un load/store sur la donnée reçue avant de passer en I. Ce load est logiquement ordonné au moment de la sérialisation de la requête originale. Condition : ce load doit être le plus ancien load en attente au moment de l'émission de la requête (**Peekaboo problem** — Table 9.2).

### 6.3 Starvation (Section 9.3.3)

**Arbitrage inéquitable :** Un cœur de priorité basse ne peut jamais accéder au bus → solution : arbitrage équitable.

**NACKs mal gérés :** Un NACK répété pour la même requête peut conduire à la starvation. Les protocoles de ce primer évitent les NACKs.

---

## 7. Points Clés pour l'Implémentation dans gem5

### 7.1 Architecture de gem5 et Garnet

gem5 implémente la cohérence de cache via la **librairie SLICC** (*Specification Language for Implementing Cache Coherence*), qui traduit des machines à états finis tabulaires (exactement comme celles du primer) en code C++. La bibliothèque **Garnet** implémente le réseau sur puce avec support des réseaux virtuels (virtual networks), indispensables pour éviter les deadlocks de protocole.

### 7.2 Ce qu'il faut retenir pour implémenter un directory

**Structure du répertoire :**
- Chaque entrée stocke l'état (I/S/M/E/O), l'identité du propriétaire, et la liste des partageurs.
- Dans gem5, l'état du répertoire est géré par le *directory controller* (machine à états distincte du cache controller).

**Types de messages :**
- **3 virtual networks** obligatoires pour éviter les deadlocks : `REQUEST_MSG`, `FORWARD_MSG` (point-to-point ordonné), `RESPONSE_MSG`.
- Les types de messages correspondent exactement à ceux du primer : GetS, GetM, PutS, PutM, Fwd-GetS, Fwd-GetM, Inv, Put-Ack, Data, Inv-Ack.

**AckCount :**
- Le répertoire calcule l'AckCount (nombre de partageurs) et l'inclut dans le message Data.
- Le cache demandeur maintient un compteur d'Acks attendus (dans son MSHR/TBE).
- L'événement `Last-Inv-Ack` déclenche la transition finale en M.

**États transitoires :**
- Encoder tous les états de la forme XY^AD dans la machine à états SLICC.
- Gérer correctement les événements reçus en état transitoire (stall, traitement, nouveaux états).

**Évictions non-silencieuses :**
- Pour le sujet du stage (Ackwise, DCC), les évictions explicites de blocs en S (PutS) sont importantes : elles permettent de maintenir à jour la liste compacte des copies.

### 7.3 Représentation compacte de la sharelist (cœur du stage)

Le sujet du stage porte sur le remplacement du répertoire classique à **presence bits O(N)** par des représentations compactes. Les mécanismes du primer directement pertinents sont :

1. **Coarse directory :** Encodage groupé des partageurs. Dans gem5, modifier la structure de l'entrée de répertoire pour utiliser un vecteur de bits groupés au lieu d'un bit par cœur.

2. **Limited pointer directory (DiriX) :** i pointeurs de log₂N bits. Modifier la logique du répertoire pour gérer les dépassements (overflow) avec la politique choisie (DiriB, DiriNB, etc.).

3. **Ackwise/DCC** (hors primer, propre au sujet de stage) : Ces protocoles encodent la liste des copies de manière encore plus compacte. L'idée clé est que la sharelist est nécessaire principalement lors des invalidations (GetM). Si on peut reconstruire la liste ou l'approcher avec peu de bits, on réduit le coût O(N) du répertoire classique.

**Points d'implémentation gem5 :**
- La structure `DirectoryEntry` dans SLICC peut être modifiée pour utiliser une représentation compacte.
- Les fonctions d'envoi d'invalidations (broadcast ou sélectif) sont à adapter selon la politique de la sharelist compacte.
- Les mécanismes d'Ack (AckCount, Inv-Ack) restent fonctionnellement identiques.
- Garnet doit avoir au moins 3 virtual networks (déjà le cas dans les configurations standard de gem5).

### 7.4 Ressources gem5 pertinentes

- Protocoles existants dans `src/mem/ruby/protocol/` : `MI_example`, `MESI_Two_Level`, `MOESI_CMP_directory` — bons exemples de référence pour la structure SLICC.
- La configuration Garnet dans `src/mem/ruby/network/garnet/` : paramétrage des virtual channels et virtual networks.
- Les MSHRs/TBEs sont implémentés dans les `CacheMemory` et `DirectoryMemory` de gem5.

---

## Synthèse

| Aspect | Snooping | Directory classique (full bit-vector) | Directory compact (sujet stage) |
|--------|----------|--------------------------------------|--------------------------------|
| Stockage état | Distribué (pas de répertoire) | O(N × M) bits | O(i × log₂N × M) bits ou moins |
| Requêtes | Broadcast ordonné | Unicast → forward | Unicast → forward |
| Transactions | 2 messages | 2–3 messages | 2–3 messages |
| Scalabilité | Faible (O(N) par requête) | Bonne | Meilleure (stockage) |
| Complexité | Modérée | Élevée (états transitoires) | Élevée + gestion overflow |
| Acquittement | Implicite (ordre bus) | Explicite (Inv-Ack + AckCount) | Explicite (identique) |

**Message central pour le stage :** Le répertoire à presence bits classique est fondamentalement correct mais coûteux en stockage (O(N) par bloc). Les protocoles Ackwise et DCC visent à réduire ce coût en représentant la sharelist de manière plus compacte, au prix d'une logique légèrement plus complexe pour gérer les cas de dépassement et reconstruire/approcher la liste lors des invalidations. L'infrastructure gem5+Garnet est parfaitement adaptée à ce type d'expérimentation : les virtual networks évitent les deadlocks, SLICC permet de spécifier les machines à états de façon lisible, et l'on peut modifier la structure `DirectoryEntry` pour expérimenter différentes représentations de la sharelist.