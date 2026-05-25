# Gotham — dossier complet

> **Document de référence exhaustif** sur le protocole Gotham, son
> implémentation dans la suite Crypto, son modèle de menace et son
> exploitation opérationnelle.
>
> **Auteurs :** Angel
> **Statut :** v0.1 implémenté + tracé ; v0.2 sur branche dédiée.
> **Licence :** AGPLv3 (open source) + commerciale (Tier-1 entreprise).
> **Compagnons :**
> - `GOTHAM.md` — RFC / spécification wire format.
> - `GOTHAM-CHECKLIST.md` — roadmap implémentation 8 tracks.
> - `SECURITY.md` — advisories actives + mitigations.
> - `PITCH.md` — pitch investisseur.
> - `crypto-gotham/HARDENING.md` — runbook fuzz + Kani + dudect.

---

## Table des matières

1. [Résumé exécutif](#1-r%C3%A9sum%C3%A9-ex%C3%A9cutif)
2. [Pourquoi Gotham existe](#2-pourquoi-gotham-existe)
3. [Fondamentaux conceptuels](#3-fondamentaux-conceptuels)
4. [Vue d'architecture](#4-vue-darchitecture)
5. [Primitives cryptographiques](#5-primitives-cryptographiques)
6. [Format paquet v0.1 — slot-based](#6-format-paquet-v01--slot-based)
7. [Format paquet v0.2 — folded-KEM](#7-format-paquet-v02--folded-kem)
8. [Payload AEAD v0.2](#8-payload-aead-v02)
9. [Sealed-sender envelope](#9-sealed-sender-envelope)
10. [Couche transport](#10-couche-transport)
11. [Le relais Gotham](#11-le-relais-gotham)
12. [Cover traffic](#12-cover-traffic)
13. [Directory authority](#13-directory-authority)
14. [Sélection de chemin](#14-s%C3%A9lection-de-chemin)
15. [Push relay mobile](#15-push-relay-mobile)
16. [Intégration dans Crypto](#16-int%C3%A9gration-dans-crypto)
17. [Threat model exhaustif](#17-threat-model-exhaustif)
18. [Comparaison avec d'autres systèmes](#18-comparaison-avec-dautres-syst%C3%A8mes)
19. [Hardening + assurance](#19-hardening--assurance)
20. [Déploiement opérationnel](#20-d%C3%A9ploiement-op%C3%A9rationnel)
21. [Roadmap technique](#21-roadmap-technique)
22. [Glossaire](#22-glossaire)
23. [Références](#23-r%C3%A9f%C3%A9rences)

---

## 1. Résumé exécutif

**Gotham** est un protocole de réseau anonyme à *latence basse*, conçu
pour transporter des messages chiffrés entre utilisateurs tout en cachant
**qui parle à qui, quand et pendant combien de temps**. Il sert de
couche transport pour l'application **Crypto** — une messagerie chiffrée
sécurité-entreprise — et est implémenté en Rust pur dans les crates
`crypto-gotham`, `crypto-gotham-relay` et `crypto-gotham-push`.

**Différences avec ses pairs** :
- Plus rapide que **Tor** (50-300 ms vs 800-2000 ms médian) car les
  délais de mixing sont calibrés pour le temps réel.
- Plus de garanties anti-corrélation que **Signal** car les métadonnées
  serveur n'existent pas (relais sans état).
- Plus simple à déployer que **Nym** car pas de chaîne de tokens, pas
  de blockchain dans la boucle.
- **Post-quantique** depuis le jour 1 (X25519 + ML-KEM-768 hybride,
  conforme NIST FIPS 203).
- Code 100% Rust, lint discipline `deny(clippy::unwrap_used)` sur les
  chemins de production.

**État actuel** :
- 8 phases sur 8 du plan d'implémentation atteintes (livrées sur la
  branche `main` + branche `gotham-v0.2`).
- 250+ tests workspace, 0 failed, 0 warning clippy.
- Pré-déploiement : aucune authority directory publique, aucun relais
  public encore hébergé.

---

## 2. Pourquoi Gotham existe

### 2.1 Le problème de la fuite métadonnée

Signal chiffre le contenu de vos messages avec **X3DH + Double Ratchet** —
une combinaison cryptographique de référence, vérifiée formellement par
plusieurs équipes académiques. Personne, pas même le serveur Signal, ne
peut lire "Alice envoie 'le projet est validé' à Bob".

Mais le serveur Signal voit ceci :
- *« Le compte associé au numéro `+33 6 ...` a envoyé 142 octets vers le
  compte associé au numéro `+33 7 ...` à 14:32:07 le 24 mai 2026. »*

Multipliez sur six mois, vous obtenez un **graphe relationnel** complet :
qui parle à qui, à quelle fréquence, à quels moments. Un analyste
compétent reconstruit l'organigramme R&D d'un industriel, détecte les
premières communications autour d'un rapprochement industriel, ou
identifie les sources d'un journaliste — **sans jamais avoir lu un seul
message**.

Pour une banque systémique, c'est un risque d'inside-trading reverse-
engineering. Pour un industriel défense, c'est une fuite d'intelligence
économique. Pour un État, c'est une perte de souveraineté.

### 2.2 Pourquoi Tor ne suffit pas

**Tor** résout largement ce problème pour le web : il cache l'adresse IP
de l'émetteur via 3 hops d'oignon-routing. Trois faiblesses pour la
messagerie temps réel :

1. **Latence trop élevée**. Tor médian sur un onion = 800-2000 ms. Trop
   pour des indicateurs de frappe ("typing..."), pour des accusés de
   lecture instantanés, pour des appels.
2. **Vulnérable à la corrélation timing** sur un Global Passive
   Adversary (GPA). Avec assez de visibilité réseau, l'observation de
   l'entrée et de la sortie corrèle les paquets par leur taille et
   leur timing.
3. **Pas de cover traffic standard**. Un utilisateur Tor inactif est
   indistinguable d'un utilisateur Tor déconnecté. Un utilisateur Tor
   actif l'est aussi. La différence binaire active/inactive fuit.

### 2.3 Pourquoi Signal ne suffit pas

Sealed-sender (Signal 2018) chiffre l'identité de l'expéditeur côté
serveur. Bonne initiative. Mais :

1. Le serveur **voit toujours** quels destinataires reçoivent des
   messages, quand, et à quelle fréquence.
2. Le serveur connaît votre *registration ID* (numéro de téléphone),
   ce qui suffit à l'identification dès qu'il y a une requête
   judiciaire (CLOUD Act).
3. Pas de mixing — la requête arrive en quelques millisecondes.
   Corrélation observation in/out triviale.

### 2.4 Pourquoi un nouveau protocole

L'idée d'un mixnet n'est pas nouvelle. **Chaum 1981** la décrit la
première fois. **Mixminion** (2002), **Loopix** (2017) et **Nym**
(2018) explorent les variantes. Mais aucun n'est :
- Conçu pour le **temps réel** (la plupart visent l'email — délais
  d'heures, voire de jours).
- Doté d'un **format paquet post-quantique** dès l'origine (Nym ajoute
  ML-KEM en cours de chemin, sur un code-base déjà ossifié).
- Distribué avec une **app utilisateur finale** prête à déployer en
  entreprise (Tchap est le seul à approcher, mais sur Matrix).

Gotham comble ce vide : Sphinx classique + Poisson Loopix-style + post-
quantique hybride + intégration native dans une app Tauri prête.

---

## 3. Fondamentaux conceptuels

### 3.1 Qu'est-ce qu'un mixnet

Un **mixnet** (réseau de mélange) est un réseau de relais qui reçoivent
des messages chiffrés, les réordonnent (les *mélangent*) sur des
délais aléatoires, et les transmettent au prochain hop. La propriété
clé : **personne ne peut faire correspondre un paquet entrant à un
paquet sortant** par observation, car (a) les paquets sont
indistinguables (taille fixe, contenu chiffré), (b) leur ordre est
randomisé, (c) du faux trafic (cover) est mélangé au vrai.

### 3.2 Le format Sphinx

**Sphinx** (Danezis-Goldberg 2009) est le format de paquet standard
pour mixnet à oignon-routing. Chaque paquet contient :

- Un **header** (`(α, β, γ)`) :
  - α = clé publique éphémère pour la couche actuelle
  - β = données de routage chiffrées par couches
  - γ = MAC qui authentifie l'header
- Un **payload** : opaque pour les hops, déchiffrable seulement au
  dernier hop.

Chaque hop fait :
1. Calcule un secret partagé `s = DH(α, sk_hop)`.
2. Dérive des sous-clés `(k_mac, k_header, k_payload, k_blind)` via HKDF.
3. Vérifie `γ ?= MAC(k_mac, header)`.
4. Décrypte la couche β qui le concerne — révèle (a) son record de
   routage (next hop addr, delay, etc.), (b) la nouvelle β à
   transmettre.
5. Re-blindit α : `α' = α^k_blind` (multiplication scalaire).
6. Transmet `(α', β', γ')` au prochain hop.

**Propriétés** :
- Un hop ne sait **rien** d'autre que son prédécesseur immédiat + son
  successeur immédiat.
- Le paquet a **taille fixe** à chaque hop (filler discipline).
- Replay détectable via cache γ (Gotham le fait : LRU + TTL 5 min).

### 3.3 Mélange Poisson + cover traffic

**Loopix** (Piotrowska et al. 2017) introduit deux ajouts critiques
sur Sphinx pour résister aux adversaires globaux :

- **Délais Poisson par hop** : chaque relais dort `Exp(λ)` ms avant
  de forwarder. Avec λ = 20-50 ms, le délai cumulé sur 3 hops reste
  100-300 ms (temps réel OK) mais introduit assez d'incertitude pour
  briser la corrélation timing.
- **Cover traffic continu** : chaque client envoie des paquets-bidons
  à des relais aléatoires, en plus de ses vrais messages. Un
  observateur ne peut plus distinguer "user actif" de "user idle".

Gotham reprend ces deux mécanismes (modules `delay.rs` et `cover.rs`).

### 3.4 Onion routing vs garlic routing

- **Onion** (Tor, Sphinx) : un message par paquet, chiffré en couches
  successives.
- **Garlic** (I2P) : plusieurs messages par paquet, regroupés et
  chiffrés ensemble.

Gotham choisit **onion** — plus simple à raisonner, format paquet bien
étudié académiquement, suffisant pour notre cas d'usage.

---

## 4. Vue d'architecture

### 4.1 Les acteurs

| Acteur | Rôle |
|---|---|
| **Sender (client)** | Construit le paquet Gotham, le ship au premier hop. App Crypto via `GothamClient`. |
| **Entry relay** | Premier hop. Reçoit le paquet du client, décrypte sa couche, transmet au mix. |
| **Mix relay(s)** | 1-3 hops intermédiaires. Découplent statistiquement entry et exit. |
| **Exit relay** | Dernier hop. Décrypte la dernière couche, livre le payload via `DeliveryHandler`. |
| **Recipient** | Destinataire final. Possède la clé pour unseal l'envelope sealed-sender. |
| **Directory authority** | Signe la liste des relais actifs (Ed25519 hors-ligne). Distribue le `SignedDirectory`. |

### 4.2 Cycle de vie d'un message

```
Alice Entry Mix Exit Bob
 │ │ │ │ │
 │ 1. sceller body │ │ │ │
 │ 2. pick path │ │ │ │
 │ 3. wrap Sphinx │ │ │ │
 │ │ │ │ │
 │── packet [α₀] ────▶│ │ │ │
 │ │ 4. unwrap │ │ │
 │ │ 5. Poisson │ │ │
 │ │── [α₁] ────▶│ │ │
 │ │ │ 4. unwrap │ │
 │ │ │ 5. Poisson │ │
 │ │ │── [α₂] ────▶│ │
 │ │ │ │ 4. unwrap │
 │ │ │ │ 6. deliver │
 │ │ │ │── body ────▶│
 │ │ 7. unseal
 │ │ 8. ratchet decrypt
 │ │ 9. emit message-received
```

À chaque hop, un observateur externe voit un paquet de **2048 octets
exactement**, encapsulé dans une trame Noise XK (+16 B de tag) au-
dessus de QUIC. Toutes les paquets sont structurellement identiques —
réels, factices, loop.

### 4.3 Les crates Rust

```
crypto/
├── crypto-gotham/ # Primitives protocole (no I/O)
│ ├── src/
│ │ ├── hybrid.rs # X25519 + ML-KEM-768 KEM
│ │ ├── header.rs # Header Sphinx v0.1 (slot-based)
│ │ ├── header_v2.rs # Header v0.2 (folded shift-and-pad)
│ │ ├── payload_v2.rs # Payload AEAD v0.2
│ │ ├── packet.rs # Wrap/unwrap paquet complet
│ │ ├── route.rs # Route descriptor
│ │ ├── relay.rs # Stateless relay state machine
│ │ ├── cover.rs # Cover scheduler Poisson
│ │ ├── directory.rs # SignedDirectory + PathSelector
│ │ ├── directory_refresh.rs # A3.7 refresh anonyme
│ │ └── sealed.rs # Sealed-sender envelope
│ ├── fuzz/ # cargo-fuzz harnesses
│ ├── proofs/ # Kani model-check
│ └── benches/timing_leak.rs # Criterion dudect-style
│
├── crypto-gotham-relay/ # Daemon QUIC + Noise XK
│ ├── src/
│ │ ├── transport.rs # QUIC + Noise XK + frame I/O
│ │ ├── process.rs # Relay::process forward/drop/deliver
│ │ ├── replay.rs # LRU + TTL 5 min cache γ
│ │ ├── delay.rs # Poisson scheduler
│ │ ├── pool.rs # Outbound connection pool
│ │ ├── client.rs # GothamClient (sender side)
│ │ ├── cover_loop.rs # Background cover task
│ │ ├── pluggable.rs # A6 trait + TLS-TCP fallback
│ │ └── main.rs # gotham-relay binary
│
└── crypto-gotham-push/ # Push notification relay
    └── src/
        ├── enrollment.rs # recipient_pk → push_token store
        ├── provider.rs # APNS + FCM trait
        ├── relay.rs # DeliveryHandler that triggers push
        └── main.rs # gotham-push-relay binary
```

---

## 5. Primitives cryptographiques

### 5.1 X25519 (RFC 7748)

**Echange de clés Elliptic-Curve Diffie-Hellman** sur la courbe
Curve25519, en représentation Montgomery.

- Sécurité classique : 128 bits.
- Sécurité post-quantique : **0** (cassable par Shor sur ordinateur
  quantique pertinent).
- Clé secrète : 32 octets clampés (`sk[0] &= 248; sk[31] &= 127; sk[31] |= 64`).
- Clé publique : 32 octets.
- Opération : `pubkey = secret * G` où G est le générateur Curve25519.

Gotham utilise X25519 pour :
- Échange éphémère par-hop (le scalaire éphémère du sender est blindé
  multiplicativement par les `k_blind` successifs).
- Identité long-terme des relais (`static_sk` du relais sert à la
  fois pour Noise XK et pour la décapsulation Sphinx).
- Sealed-sender (clé éphémère par message + clé long-terme du
  destinataire).

Crate Rust : `x25519-dalek 2.x` (avec feature `static_secrets`).

### 5.2 ML-KEM-768 (NIST FIPS 203)

**Module-Lattice Key Encapsulation Mechanism**, finaliste NIST PQ
2024. Sécurité post-quantique catégorie 3 (= ~192 bits classiques).

- Clé secrète : 2400 octets.
- Clé publique : 1184 octets.
- Texte chiffré (ciphertext) : 1088 octets.
- Secret partagé : 32 octets.

Gotham l'utilise dans le **hybride** : chaque échange par-hop combine
X25519 (rapide, prouvé) et ML-KEM-768 (post-quantique mais lourd).
Le secret partagé final = `HKDF(X25519_shared || MLKEM_shared)`.

**Pourquoi l'hybride** : si l'un des deux algorithmes tombe (X25519
cassé par adversaire quantique, ML-KEM cassé par avancée
cryptographique classique), l'autre tient encore.

Crate Rust : `ml-kem 0.2` (implémentation pure Rust, conforme FIPS 203).

**Caveat v0.1** : le format paquet 384 B ne contient pas le ciphertext
ML-KEM (1088 B trop volumineux). L'hybride PQ vit dans
`crypto-gotham/src/hybrid.rs` mais **n'est pas wired** dans le format
paquet v0.1 ; le wire utilise X25519 seul. v0.2 introduira un format
"folded-KEM" qui peut potentiellement intégrer ML-KEM via stockage
hors-header (à confirmer en design v0.2 final).

### 5.3 HKDF-SHA256 (RFC 5869)

**HMAC-based Key Derivation Function** avec SHA-256. Utilisé partout
dans Gotham pour dériver des sous-clés à partir d'un secret partagé.

```text
HKDF(salt, IKM, info, length) → OKM
```

Gotham :
- `derive_hop_subkeys(shared_x)` : un shared 32 B → 4 sous-clés 32 B
  (`k_mac, k_header, k_payload, k_blind`) avec salt `"gotham-hop-v1"` et
  info `"gotham:subkeys"`.
- `sealed::seal` : ephem-DH → 32 B `k_seal` avec salt `"gotham-sealed-v1"`,
  info `"k_seal"`.

### 5.4 ChaCha20 (RFC 8439)

**Stream cipher** 256-bit key, 96-bit nonce. Conçu par Bernstein
2008, performance constante-time même sur CPU sans AES-NI.

Gotham :
- ChaCha20 seul pour la couche β du header (nonce zéro, sûr car key
  est per-packet ephemeral).
- ChaCha20-Poly1305 (AEAD) pour la couche payload + sealed-sender.

### 5.5 Poly1305 (RFC 8439)

**One-time MAC** complémentaire à ChaCha20. 128-bit tag, key 256-bit.

Gotham :
- Authentifie γ dans le header Sphinx (`Poly1305(k_mac, header_meta)`).
- Composé avec ChaCha20 en AEAD pour les payloads.

### 5.6 Ed25519 (RFC 8032)

**Signatures EdDSA** sur Curve25519 (twisted Edwards). Déterministes,
malléabilité prouvée nulle, side-channel-safe.

Gotham :
- Signature de la `SignedDirectory` par l'authority.
- Signature des audit logs entreprise (hors Gotham mais même primitive).

Crate Rust : `ed25519-dalek 2.x`.

### 5.7 SHA-256

Hash 256-bit, standard NIST. Utilisé dans HKDF, dans HMAC, dans les
fingerprints d'identité.

### 5.8 BLAKE3

Hash plus rapide que SHA-2, conçu par Aumasson et al. 2020. Gotham
l'utilise pour quelques fingerprints internes mais privilégie SHA-256
pour la conformité FIPS dans les chemins critiques.

### 5.9 Argon2id (RFC 9106)

**Memory-hard password hash**, vainqueur du Password Hashing
Competition 2015. Utilisé par Crypto (pas par Gotham directement) pour
dériver la clé maîtresse SQLCipher à partir du mot de passe utilisateur.

Paramètres 2026 : `m=46 MiB, t=1, p=1` (interactive) ; `m=64-256 MiB,
t=2-4, p=4` (offline-tolerant).

### 5.10 Noise XK

**Pattern de protocole Noise** : eXtended Knowledge des deux côtés.

- Client connaît la pubkey du serveur à l'avance (pinned).
- Serveur ne connaît pas l'identité client (anonyme).
- 3 messages handshake : `<- e, es | -> e, ee | <- s, se`.

Gotham utilise `Noise_XK_25519_ChaChaPoly_BLAKE2s` pour le canal de
transport par-link entre relais, et entre client et premier hop. Le
client se ré-identifie via une clé X25519 éphémère par-connexion ;
le relais s'identifie via sa `static_sk` long-terme advertised dans le
directory.

Crate Rust : `snow 0.9`.

---

## 6. Format paquet v0.1 — slot-based

### 6.1 Constantes

| Symbole | Valeur | Rôle |
|---|---|---|
| `PACKET_SIZE` | 2048 octets | Taille totale du paquet (fixe à chaque hop) |
| `HEADER_LEN` | 384 octets | Taille de l'header |
| `PAYLOAD_SIZE` | 1664 octets | `PACKET_SIZE - HEADER_LEN` |
| `MAX_HOPS` | 5 | Profondeur maximale du chemin |
| `ALPHA_LEN` | 32 octets | Clé publique X25519 éphémère |
| `BETA_LEN` | 320 octets | Routing block (5 × 64) |
| `GAMMA_LEN` | 16 octets | MAC Poly1305 |
| `TRAILER_LEN` | 12 octets | Padding aléatoire couvert par γ |
| `RECORD_LEN` | 64 octets | Une routing record par hop |

### 6.2 Layout binaire

```text
offset size field
    0 1 version (= 1)
    1 1 mode (0=low-latency, 1=balanced, 2=paranoid)
    2 1 hop_count (n; 1 ≤ n ≤ MAX_HOPS)
    3 1 hop_index (i; 0 ≤ i < hop_count) — incrémente à chaque hop
    4 32 α — X25519 ephemeral public key for this hop
   36 320 β — 5 × 64 B encrypted routing slots
  356 16 γ — Poly1305 MAC over (meta || α || β[slot_i] || trailer)
  372 12 trailer — random padding (covered by γ)
```

### 6.3 Routing record (64 B)

```text
offset size field
    0 4 next_ipv4
    4 2 next_port (big-endian)
    6 32 next_node_id (relay identity fingerprint / X25519 pubkey)
   38 16 next_gamma (γ que le prochain hop installera dans son header)
   54 4 delay_micros (big-endian) — Poisson sample en microsec
   58 1 flag (bit 0 = IS_LAST_HOP, bit 1 = DELIVER_LOCAL)
   59 5 _padding (doit être zéro)
```

### 6.4 Construction côté sender

1. **Path selection** via `PathSelector::pick(rng, hop_count)` —
   diversifie par operator + /16 IPv4.
2. **Derive chain** : sample un scalaire éphémère `s` ; pour chaque
   hop i : `α_i = α_{i-1}^{k_blind_{i-1}}` ; `s_i = X25519(s, pk_i)`
   ; `sub_keys_i = HKDF(s_i)`.
3. **Build records** : record i carry `next_addr, next_port,
   next_node_id, next_gamma` du hop i+1. Last hop : `flag |=
   IS_LAST_HOP`.
4. **Init β** : `β = random(BETA_LEN)`. Les slots non-utilisés
   restent aléatoires (indistinguables des slots chiffrés).
5. **Encrypt slots inside-out** :
   ```
   next_gamma = 0
   for i from n-1 downto 0:
       record_i.next_gamma = next_gamma
       β[i*64..(i+1)*64] = record_i.encode() XOR ChaCha20(k_header_i)
       γ_i = Poly1305(k_mac_i, meta || α_i || β[i*64..(i+1)*64] || trailer)
       next_gamma = γ_i
   ```
   Après la boucle, `next_gamma` contient γ_0 — le MAC du premier hop.
6. **Assemble** : packet = `header.encode() || payload || zero-pad`.

### 6.5 Unwrap côté hop

1. **Compute shared** : `s = X25519(my_sk, α)`.
2. **Derive sub_keys** : `HKDF(s) → (k_mac, k_header, k_payload, k_blind)`.
3. **Verify γ** : `Poly1305(k_mac, meta || α || β[hop_index * 64 ..]
   || trailer) ?= γ`. Si non → drop (`BadMac`).
4. **Decrypt slot** : `record = β[hop_index*64 ..] XOR ChaCha20(k_header)`.
5. **Re-blind α** : `α' = X25519(k_blind, α)` (multiplication scalaire).
6. **Build next header** : `(α', β, record.next_gamma, hop_index + 1)`.
   `β` reste **inchangé** (les autres slots restent aléatoires/chiffrés).
7. **Decision** : si `record.is_last_hop()` → `DeliverLocal { payload }`.
   Sinon → `Forward { next_addr, packet }`.

### 6.6 Propriétés et caveats v0.1

| Propriété | Statut |
|---|---|
| Confidentialité content (sealed-sender layer) | |
| Confidentialité par-hop (un hop ne peut lire que sa propre slot) | |
| Anti-replay (cache γ LRU + 5 min TTL) | |
| Anti-tagging (γ MAC chain) | |
| Position-hiding (hop_index ne fuit pas la position dans le chemin) | — le byte `hop_index` est visible (1 B fuite) |
| Length-hiding | — paquets fixes 2048 B |
| Post-quantique au niveau header | — X25519 seul dans v0.1 wire |

Le compromis "1 B fuite" était volontaire pour v0.1 — simpler implementation,
testé en condition réelle avant d'introduire le folded design plus
complexe. v0.2 supprime cette fuite.

---

## 7. Format paquet v0.2 — folded-KEM

Implémenté sur la branche `gotham-v0.2`, module `header_v2.rs`.

### 7.1 Différences clés vs v0.1

| Aspect | v0.1 | v0.2 |
|---|---|---|
| `hop_index` byte | présent | **supprimé** |
| `hop_count` byte | présent | **supprimé** |
| β layout | slots fixes `[slot_0 \| slot_1 \| … \| slot_{MAX-1}]` | buffer rotatif (current hop toujours à offset 0) |
| MAC scope | par-slot uniquement | β complète + meta |
| Position-hiding | brisée dès le premier relai | structurelle |
| Total header size | 384 B (inchangé) | 384 B |
| Wire compat | breaking change (version byte 1 → 2) | breaking change |

### 7.2 Layout v0.2

```text
offset size field
    0 1 version (= 2)
    1 1 mode
    2 2 reserved (doivent être zéro)
    4 32 α
   36 320 β — folded routing block
  356 16 γ — Poly1305 MAC sur (meta || α || β || trailer)
  372 12 trailer
```

### 7.3 Algorithme "shift-and-pad" classique Sphinx

À chaque hop, β se comporte comme un buffer circulaire :

1. Le hop décrypte β via XOR avec `ChaCha20(k_header)`.
2. Lit ses `RECORD_LEN = 64` premiers octets → c'est son record.
3. Décale β à gauche de 64 octets.
4. Padde la fin avec 64 octets dérivés déterministiquement du stream
   ChaCha20 du hop (`ρ[BETA_LEN .. BETA_LEN + RECORD_LEN]`).

**Côté sender, la construction est récursive** :

```text
# Step 1: cumulative filler (longueur (n-1) * RECORD_LEN)
filler = []
for i in 0..n-1:
    ρ_i = ChaCha20(k_header_i, len = BETA_LEN + RECORD_LEN)
    cur_len = len(filler)
    filler += [0] * RECORD_LEN
    filler XOR= ρ_i[BETA_LEN - cur_len .. BETA_LEN + RECORD_LEN]

# Step 2: innermost β
β = (record_{n-1} || random_pad) XOR ρ_{n-1}[0 .. inner_len] || filler

# Step 3: onion-wrap
for i in n-2 downto 0:
    β = (record_i || β[0 .. BETA_LEN - RECORD_LEN]) XOR ρ_i[0 .. BETA_LEN]
    γ_i = Poly1305(k_mac_i, meta || α_i || β || trailer)
    record_{i-1}.next_gamma = γ_i

return (α_0, β, γ_0)
```

L'invariant magique : grâce à la construction du filler, après que le
hop *i* a décalé-et-paddé β, le résultat est **byte-identique** à ce
que le sender avait calculé pour le hop *i+1*. Une vérification γ
chez le hop *i+1* passe donc proprement.

### 7.4 Garanties supplémentaires v0.2

- **Position-hiding structurelle** : un observateur (ou un relai
  malveillant) ne peut PAS distinguer le hop 0 du hop 4 par
  inspection du paquet.
- **MAC scope complet** : tout pli sur β est détecté immédiatement
  (v0.1 ne MAC-pas que le slot courant).
- **Format plus proche du Sphinx académique** : facilite la
  vérification formelle ultérieure.

### 7.5 État

- Module `header_v2.rs` : **10/10 tests** passing.
- Algorithme implémenté, encode/decode round-trip vérifié, rejection
  sur tampering β/γ couverte.
- **Pas encore wired** dans `packet.rs` ni dans `process.rs` — gate
  sur l'audit externe (Quarkslab / Synacktiv) avant promotion en
  default.

---

## 8. Payload AEAD v0.2

Module `payload_v2.rs` sur la branche `gotham-v0.2`.

### 8.1 Conception

v0.1 ne protège pas la **région payload** (1664 octets après le
header) — seul le sealed-sender envelope qui vit dedans est chiffré.
Un relais intermédiaire malveillant peut **tagger** un paquet
(flipper un byte) pour marquer un message vers un exit
ISP-cooperant. Le sealed-sender détecterait le tampering au
destinataire, mais à ce moment-là la corrélation
"paquet-tampé-entré-ici, message-arrivé-là" est déjà observable.

v0.2 ajoute une couche **AEAD au dernier hop** : `ChaCha20-Poly1305`
sous `k_payload_{n-1}` (la clé payload dérivée pour le dernier
hop). Tampering en milieu de chemin → échec AEAD au dernier hop →
drop silencieux. Le paquet ne quitte JAMAIS le mixnet vers le
destinataire si tampered.

### 8.2 Wire layout (constant à chaque hop)

```text
offset size field
     0 1648 ciphertext (AEAD-encrypted under k_payload_{n-1})
  1648 16 Poly1305 tag
                 total = 1664 = PAYLOAD_SIZE
```

### 8.3 Plaintext format

```text
offset size field
     0 4 body_len (BE u32, ≤ BODY_CAP = 1644)
     4 L body (sealed-sender envelope OR arbitrary bytes)
   4+L ... zero pad → PER_HOP_PAYLOAD = 1648
```

### 8.4 Pourquoi pas LIONESS (per-hop onion)

La construction classique Sphinx-payload utilise LIONESS — une wide-
block cipher 4-rounds Luby-Rackoff qui permet AEAD per-hop sans
shrinking ciphertext. Implementation correcte demande discipline
filler analogue au header. **v0.2 ship une version simplifiée
"last-hop-only"** qui est :
- Plus simple à auditer (un seul appel AEAD).
- Suffisant pour la propriété "tagging detected end-to-end".
- Tracé pour v0.3 → upgrade vers LIONESS once code-base stable.

### 8.5 État

- 10/10 tests pour `payload_v2`. Round-trip à 1, 3 et 5 hops, tampering
  detection, framing length-prefix.
- Wiring dans `client.rs`/`process.rs` gate sur audit (idem header_v2).

---

## 9. Sealed-sender envelope

Module `crypto-gotham/src/sealed.rs`. Indépendant des changements v0.2,
fonctionne avec v0.1 et v0.2.

### 9.1 Objectif

Cacher l'**identité de l'émetteur** même au relais de sortie + au
destinataire (tant qu'il n'a pas la clé pour unseal). Inspiration :
Signal sealed-sender.

### 9.2 Wire format (60 B overhead)

```text
offset size field
    0 32 ephemeral X25519 public key (per-message)
   32 12 ChaCha20-Poly1305 nonce
   44 32 sender identity X25519 public key ┐ AEAD-encrypted
   76 L inner body (e.g. Double-Ratchet CT) ┤ under k_seal
   ... 16 Poly1305 tag ┘
```

`k_seal = HKDF-SHA256(X25519(ephem_sk, recipient_pk), "gotham-sealed-v1")`.

### 9.3 Forward secrecy

La keypair X25519 éphémère est générée fresh par message, puis
**droppée immédiatement** après calcul du secret partagé. Un attaquant
qui compromet la clé long-terme de l'émetteur **après envoi** ne peut
PAS retroactively décrypter les messages — il lui faudrait l'éphémère,
qui n'a jamais existé en mémoire persistante.

### 9.4 Wiring dans Gotham

`GothamClient::send_sealed(rng, relays, hop_count, recipient_pk,
sender_pk, body)` :

1. `envelope = sealed::seal(rng, recipient_pk, sender_pk, body)`.
2. Frame the envelope avec un préfixe length 4 B big-endian (pour
   que le destinataire puisse trouver sa fin dans la région payload
   zero-paddée).
3. `self.send(rng, relays, hop_count, &framed)` — wrap Sphinx normal
   par-dessus.

Côté relais de sortie : `make_unsealing_delivery_handler(my_sk,
inner)` retourne une `DeliveryHandler` qui parse le préfixe,
appelle `unseal`, et passe `(sender_pk, body)` à `inner`. Sur unseal
fail (wrong recipient, tampering, malformation) → drop silencieux,
pas de log avec le contenu.

---

## 10. Couche transport

### 10.1 Stack par-link

```text
Gotham packet (2048 B fixed)
  │
  ▼
Noise XK (snow) — per-link symmetric ChaCha20-Poly1305
  │ + 16 B AEAD tag → 2064 B on the wire
  ▼
QUIC bi-stream over TLS 1.3 (rustls)
  │ TLS cert self-signed ; Noise XK provides real authentication
  │ custom rustls verifier accepts any cert
  ▼
UDP (default port 443)
```

### 10.2 Pourquoi deux couches crypto

- **QUIC + TLS 1.3** = streams fiables, 0-RTT resumption, congestion
  control moderne, NAT traversal, paquets indistinguables de HTTPS
  vanilla sur le wire (DPI-résistance gratuite).
- **Noise XK** = pin la pubkey long-terme du relais (la même que celle
  advertisée dans le directory) + prévient les attaques TLS-cert-MITM.
  Le client est anonyme à la couche Noise (pattern XK : client pas
  authentifié côté Noise — l'authentification client se fait à la
  couche Gotham via la chaîne α).

### 10.3 Pool de connexions

Module `pool.rs`. Sans pool, chaque `forward_packet` paie :
- Handshake QUIC (~1 RTT)
- Handshake Noise XK (3 messages = ~1.5 RTT)

À 1 paquet/sec/conversation et 200 ms one-way, ça fait 60-80%
d'overhead. Le pool key sur `(addr, peer_pubkey)`, réutilise une
unique `(quinn::Connection, SendStream, TransportState)` par couple,
amortissant le handshake à un one-time cost.

Éviction LRU au max size (default 128). Détection connexion morte →
retire de pool + retry avec fresh connection. v0.2 ajoutera un sweep
background pour proactive evict idle > 5 min.

### 10.4 Pluggable transports (A6)

Module `pluggable.rs`. Trait `GothamTransport` :
```rust
async fn connect(&self, addr) -> Result<Box<dyn ConnLike>>;
async fn probe(&self) -> bool;
fn name(&self) -> &'static str;
```

Implémentations :

| Variante | Statut | Notes |
|---|---|---|
| **QuicTransport** (UDP/443) | par défaut | Le code existant `transport.rs` wrappé. |
| **TlsTcpTransport** (TCP/443) | live | Vanilla HTTPS fingerprint (h2 + http/1.1 ALPN), SNI rotatif (`cdn.cloudflare.com` / `s3.amazonaws.com` / EU enterprise hosts). |
| **Obfs4Transport** | scaffold | Trait shape fixé, body TODO — nécessite intégration `obfs4-rs` ou port clean-room du spec Yawning Angel. |
| **MeekCdnTransport** | scaffold | HTTPS POST domain-fronté via Cloudfront / Fastly. Trait shape fixé. |

**AdaptiveTransport** : sélecteur best-of probe-and-stick. Premier
transport de la liste = hot path (QUIC par défaut). Sur `failure_
threshold` (default 3) échecs consécutifs → demote et walk down la
liste. Stratégie de sélection identique Tor PT3.

---

## 11. Le relais Gotham

Binaire `gotham-relay` (`crypto-gotham-relay/src/main.rs`).

### 11.1 Stateless design

Le relais n'a **aucun état persistant** au-delà :
- Sa clé long-terme X25519 (lue depuis le fichier au boot).
- Une in-memory replay cache (perdue au restart — pas de DoS sur la
  durée).
- Le pool de connexions outbound (idem).

Pas de queue de messages, pas de SQLite, **rien à saisir judiciairement**.

### 11.2 Replay cache

Module `replay.rs`. Cache LRU borné + TTL 5 min sur les γ vus. Chaque
paquet entrant est vérifié contre le cache :
- Miss → ajouter γ, processer normalement.
- Hit → drop silencieux (replay attack).

Capacity par default : 16 384 entrées. TTL 300 s. Configurable via CLI.

### 11.3 Poisson scheduler

Module `delay.rs`. `PoissonScheduler::next_delay(rng)` retourne un
sample `Exp(1/λ_micros)`. Application au paquet : `tokio::sleep(delay)`
avant forward.

Tests : sample-mean validation sur 10 000 samples avec tolérance ±5%
sur la mean attendue.

### 11.4 Process function

Module `process.rs`. Fonction pure :

```rust
fn process(&mut self, rng: &mut Rng, packet: &[u8]) -> ProcessOutcome
```

`ProcessOutcome` ∈ :
- `Drop(DropReason)` — bad MAC, replay, malformed.
- `DeliverLocal { delay, payload }` — last hop, hand to delivery handler.
- `Forward { next_addr, next_node_id, delay, packet }` — onion-peeled
  packet to forward after `delay`.

Stateless en dehors du replay cache → testable en isolation, fuzz-
friendly, model-checkable par Kani.

### 11.5 Service systemd hardened

`crypto-gotham-relay/deploy/gotham-relay.service` :

- `User=gotham` (compte non-privilégié dédié)
- `NoNewPrivileges=true`
- `ProtectSystem=strict` + `ProtectHome=true`
- `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX`
- `MemoryDenyWriteExecute=true`
- `SystemCallFilter=@system-service`
- `CapabilityBoundingSet=` (vide)
- `PrivateDevices=true`
- `RestrictNamespaces=true`
- `LockPersonality=true`

### 11.6 Monitoring (privacy-preserving)

Le relais expose des **counters Prometheus uniquement**, jamais de
per-packet data, jamais d'IPs. Métriques exposées :
- `gotham_relay_packets_received_total`
- `gotham_relay_packets_forwarded_total`
- `gotham_relay_packets_dropped_total{reason}` (reason ∈ bad_mac,
  replay, malformed, self_loop)
- `gotham_relay_packets_delivered_total`
- `gotham_relay_pool_size`
- `gotham_relay_uptime_seconds`

---

## 12. Cover traffic

Module `crypto-gotham/src/cover.rs` + `crypto-gotham-relay/src/cover_loop.rs`.

### 12.1 CoverScheduler Poisson

Background task qui émet des paquets à des intervalles
`Exp(1/λ_mode)`. Trois modes :

| Mode | λ (paquets/sec) | Cadence | Latence ajoutée |
|---|---|---|---|
| **Paranoid** | 1/5 | 1 paquet toutes les 5 s | mass |
| **Balanced** | 1/30 | 1 toutes les 30 s | par-défaut |
| **EcoMobile** | 1/120 | 1 toutes les 2 min | discrètement actif |

À chaque tick :
1. Lit la queue de vrais messages en attente.
2. Décide selon le mode + l'état queue : `Real`, `Drop`, `Loop`.

### 12.2 Drop vs Loop vs Real

| Intent | Comportement |
|---|---|
| **Real** | Pop le vrai message de la queue, ship via `GothamClient::send`. |
| **Drop** | Ship un paquet de remplissage vers un "sink relay" — drop silencieux côté sink. |
| **Loop** | Ship un paquet vers self via 3 hops — confirme que le réseau est UP, fournit du cover. v0.1 implémenté comme Drop (pas de self-registration directory). |

Indistinguabilité au wire : toutes les paquets sont 2048 B, Noise-
encryptés, chiffrés par hop. Un observateur ne peut PAS distinguer
Real / Drop / Loop.

### 12.3 Battery-aware degradation

`CoverMode::battery_adjusted(battery_pct, is_charging)` :
- Si `battery_pct < 20` AND `!is_charging` → demote au mode inférieur.
- Si `battery_pct < 10` AND `!is_charging` → demote à `EcoMobile`.
- En charge ou ≥ 20% → mode user-configured respecté.

Évite de tuer la batterie smartphone tout en préservant un minimum
de cover (le minimum eco continue à émettre).

---

## 13. Directory authority

Module `crypto-gotham/src/directory.rs`.

### 13.1 RelayDescriptor

```rust
struct RelayDescriptor {
    id_pubkey_hex: String, // X25519 identity (= kem in v0.1)
    kem_pubkey_hex: String,
    addr: String, // "ip:port"
    tier: RelayTier, // Entry / Mix / Exit / Mirror
    country: Option<String>, // ISO 3166-1 alpha-2
    asn: Option<u32>,
    operator: Option<String>, // pour la diversité
    uptime_pct: Option<f32>,
}
```

### 13.2 SignedDirectory

```rust
struct DirectoryDoc {
    version: u8, // = DIRECTORY_VERSION (1)
    valid_after: u64, // Unix seconds
    valid_until: u64,
    relays: Vec<RelayDescriptor>, // sorted by id_pubkey_hex (canonical)
}

struct SignedDirectory {
    doc: DirectoryDoc,
    authority_pubkey_hex: String,
    signature_hex: String, // Ed25519 over canonical_bytes(doc)
}
```

Signature : `serde_json::to_vec(&doc)` (compact, deterministic) → Ed25519
sign. Verification : strict authority pubkey check + signature verify
+ schema version match + validity window check (now ≥ valid_after AND
now ≤ valid_until).

### 13.3 Distribution v0.1

Le `SignedDirectory` peut être distribué via :
- HTTPS clearnet (avec mitigation IP leak via Tor SOCKS).
- Bundle au déploiement (config fichier).
- **Refresh anonyme via Gotham lui-même** (module
  `directory_refresh.rs`, A3.7 — voir §13.4).

### 13.4 Refresh anonyme (A3.7)

Sur la branche `gotham-v0.2`. Le client envoie une `DirectoryRequest`
à un relais tagué "mirror" via Gotham normal. Le mirror répond via
Gotham également (model A : chaque user = relai, donc le requester a
une stable Gotham address pour le retour ; pas de SURB requis en
v0.1).

Wire format inside la sealed envelope body :
```text
byte 0 type tag (1 = Request, 2 = Response)
byte 1.. serde-JSON encoded
```

Chunking pour grandes directories (> ~1.4 kB par paquet) géré par
`chunk_response` / `reassemble`.

### 13.5 Authority management

Clé Ed25519 de l'authority **stockée hors-ligne sur YubiKey**.
Refresh quotidien : opérateur signe une nouvelle directory avec
valid_after = now, valid_until = now + 24h, publie.

v0.2 : N-of-M multi-sig directory (plusieurs authorities cosignées,
M autorités requises pour valider). Tracé dans la spec, pas
implémenté.

---

## 14. Sélection de chemin

`PathSelector::pick(rng, hop_count)` dans `directory.rs`.

### 14.1 Contraintes de diversité

Pour chaque chemin sélectionné :
- **Tier diversity** : 1 Entry + N-2 Mix + 1 Exit (les Mirror sont
  utilisables comme Mix).
- **Operator diversity** : `consecutive_diverse` — deux hops successifs
  ne peuvent pas avoir le même `operator` string.
- **AS diversity (/16 IPv4)** : deux hops successifs ne peuvent pas
  être dans le même /16. Exception **loopback** (127.0.0.0/8) pour
  permettre le mode dev 3-relais.

### 14.2 Hop count per mode

| Mode | Hops | Latence median attendue |
|---|---|---|
| `low_latency` | 3 | 50-150 ms |
| `balanced` | 4 | 100-250 ms |
| `paranoid` | 5 | 150-300 ms |

Le sender peut override le mode global avec un `hop_override` lors du
`gotham_send`.

### 14.3 Diversité géographique (v0.2)

Tracé pour v0.2 : ajouter contraintes `country` (éviter chemins
all-FR ou all-DE) pour résister à la coercition juridictionnelle
d'un seul État.

---

## 15. Push relay mobile

Crate `crypto-gotham-push`, binaire `gotham-push-relay`.

### 15.1 Problème mobile

App backgrounded sur iOS / Android = pas de connexion QUIC persistante
possible (batterie + restrictions OS). Comment réveiller l'app à
l'arrivée d'un message ?

Réponse : **APNS / FCM** (Apple Push Notification Service, Firebase
Cloud Messaging). Le relais push connaît le token de l'utilisateur
mobile, envoie une notification "wake" minimale, l'app se réveille,
ouvre une connexion Gotham brève, pull les messages, se rendort.

### 15.2 Threat model addendum

Le push relay est l'entité **la plus sensible** du pipeline mobile.
Compromission = connaissance de la mapping `recipient_pk → push token`
+ timing de chaque message entrant. Mitigations :

1. **Push tokens chiffrés au rest** dans `EnrollmentStore` (caller-
   supplied master key, typiquement YubiKey-backed). Cleartext ne
   vit qu'en RAM le temps de la RPC APNS/FCM.
2. **Pas de log per-delivery** du recipient_pk. Counters only.
3. **Payload APNS/FCM minimal** : `{ kind: "wake", message_count: N }`.
   Le contenu réel est fetché séparément par l'app via Gotham. Le
   push relay **ne voit jamais** le plaintext message.
4. **Rotation token 7 jours + 24h grace window**. Re-enrollment
   forcé à expiration.

### 15.3 Architecture

- `EnrollmentStore` trait — mémoire + snapshot JSON, ou Postgres en
  prod.
- `PushProvider` trait — `ApnsProvider` (p8 JWT-signed HTTP/2),
  `FcmProvider` (REST + OAuth2 service-account).
- `make_push_delivery_handler` compose avec `make_unsealing_delivery_
  handler` : le relai push est juste un Gotham exit normal + un
  callback APNS/FCM.

### 15.4 État

- Binaire `gotham-push-relay` runnable (`--help` fonctionnel).
- Eviction periodic loop (1h).
- 11/11 tests (enrollment + provider + handler smoke).
- APNS HTTP/2 hot path : TODO (nécessite intégration `a2` ou `hyper-h2`
  + ES256 JWT signing).
- FCM REST hot path : TODO (nécessite intégration `reqwest` + OAuth2
  service-account token mint).
- Admin HTTP endpoint pour enrollment out-of-band : TODO.

---

## 16. Intégration dans Crypto

### 16.1 Model A : embedded relay

Choix v0.1 : **chaque utilisateur Crypto embarque son propre relais
Gotham** dans l'app. Le relais embarqué est l'**exit hop** pour les
messages destinés à cet utilisateur, donc inbound delivery est in-
process via mpsc — pas de subscribe protocol externe.

Implications :
- + Réutilise tout le code Gotham existant sans nouveau wire.
- + Inbound naturel via `DeliveryHandler` → drainer → `SessionManager`.
- − Nécessite IP publiquement reachable (NAT traversal en v0.2).
- − Pas adapté grand-public sans relai-hosting hosted.

Model B (subscribe protocol pour users derrière NAT) est tracé pour
v0.2.

### 16.2 Identité Gotham

Module `crypto-tauri/src-tauri/src/gotham.rs` :

```rust
pub fn load_or_create_x25519_identity(gotham_dir) -> [u8; 32]
```

Persiste `<gotham_dir>/identity.key` (chmod 0600 sur Unix). Génération
à la première init via `OsRng`, clamping X25519, écriture atomic.

### 16.3 Schema v15 — gotham_pk_hex

Migration crypto-store v15 (additive-only) :

```sql
ALTER TABLE contacts ADD COLUMN gotham_pk_hex TEXT;
ALTER TABLE own_profile ADD COLUMN gotham_pk_hex TEXT;
```

NULL sur les rows existantes → fallback transparent au SMP path
pour les contacts pré-Gotham. Helpers CRUD :
- `get_contact_gotham_pk(name)`
- `set_contact_gotham_pk(name, pk_hex)`
- `find_contact_by_gotham_pk(pk_hex)` — index inverse pour le drainer.
- `get_own_gotham_pk()` / `set_own_gotham_pk(pk_hex)`.

### 16.4 Commandes Tauri exposées

| Commande | Rôle |
|---|---|
| `gotham_init` | Idempotent. Load/gen identity, start embedded relay, build/load dev directory, construct `GothamClient`, spawn drainer. Émet `gotham-init-progress` events. |
| `gotham_status` | Read-only snapshot pour UI. |
| `gotham_send(recipient_pk, body)` | Seal + ship via 3-hop. |
| `gotham_devmode_enable` | Spawn 2 satellites + persiste 3-relay self-loop directory. Démo end-to-end mono-machine. |
| `gotham_set_contact_pk(contact, pk_hex)` | V0.1 manuel : paste hex out-of-band reçu du peer. |
| `gotham_get_contact_pk(contact)` | Lookup pour UI. |
| `gotham_get_my_pk` | Display pour partage out-of-band. |
| `gotham_set_enabled(bool)` | Toggle setting `gotham_transport_enabled`. |
| `gotham_get_enabled` | Read setting. |

### 16.5 Drainer inbound

Background tokio task spawnée à `gotham_init` :

1. Drain `inbox_rx` (mpsc des deliveries unsealed).
2. Pour chaque `(sender_pk, body)` :
   - Lookup `find_contact_by_gotham_pk(sender_pk)` → contact name (ou
     emit `gotham-inbound-unknown`).
   - Parse `MSG_TYPE_ENCRYPTED` + decrypt via `SessionManager`.
   - Dispatch par `ChatPayload` variant : `Text`, `ProfileUpdate`,
     `ReadReceipt` (autres en TODO).
   - Persiste via `save_message` + ratchet update.
   - Emit `message-received` Tauri event au frontend.

### 16.6 send_message fast-path

Dans `send_message` (lib.rs:996+) :

```rust
let gotham_recipient_pk_hex = {
    let enabled = get_setting("gotham_transport_enabled") == "1";
    if enabled { get_contact_gotham_pk(contact) } else { None }
};

if let Some(recipient_hex) = gotham_recipient_pk_hex {
    // Try Gotham first
    let result = gotham_client.send_sealed(...).await;
    if result.is_ok() {
        // Mark sent, return.
        return Ok(());
    }
    // Else fall through to SMP path.
}

// Existing SMP path (onion-or-direct over Tor) unchanged.
```

→ Backward compat transparente. Si le user désactive Gotham, code path
inchangé pré-A4.

### 16.7 UI exposée

`Settings → Network → Anonymous transport` : 3 radios :
- **Tor** — onion routing (v0.1, default)
- **Lokinet** — TUN-based anonymous overlay
- **Gotham mixnet** — post-quantum, metadata-hiding (v0.2)

Chaque radio porte un sous-texte explicatif (threat model + caveat)
**directement dans l'UI** pour que les screenshots DSI soient auto-
documenteurs.

`Settings → Profile → Gotham mixnet (preview)` : section deep-config :
- Status (local relay addr, directory entries, my PK hex)
- Bootstrap progress bar (driven par `gotham-init-progress` events)
- `Initialize Gotham` button
- "Enable 3-relay self-loop" (dev mode)
- Paste box pour set un contact gotham_pk

`Identity` section : bannière "Gotham mixnet — post-quantum, metadata-
hiding (SMP/Tor fallback)" affichée when `transport === "gotham"`.

---

## 17. Threat model exhaustif

### 17.1 Adversaires considérés

| Adversaire | Capacités | Résistance Gotham |
|---|---|---|
| **Passif réseau local** (ISP, café WiFi) | Lit le trafic du user | tout chiffré, paquets fixes 2048B, mixing Poisson |
| **Compromis d'un seul relais** (n'importe quel tier) | Lit le trafic qu'il traite | ne voit que `prev_hop`, `next_hop`, son record décrypté |
| **Replay attaquant** | Réinjecte des paquets capturés | cache γ LRU + 5 min TTL |
| **Tagging attaquant** | Flip un byte pour marquer un paquet | γ MAC chain break à n'importe quel hop ; v0.2 ajoute AEAD payload au dernier hop |
| **Adversaire quantique futur** | Casse les ECC clavier | v0.1 X25519 seul fragile ; v0.2 hybride ML-KEM-768 |
| **Coercition juridictionnelle d'un opérateur** | Force un opérateur à logger | mitigé par tier-diversity + multi-pays operator policy (encore non-déployé) |
| **DPI / blocage UDP 443** | Block QUIC | A6 pluggable transports — fallback TLS-TCP/443, obfs4 et meek-CDN à venir |
| **Compromis app endpoint** | Malware sur device user | hors scope Gotham (mitigé par hardening OS + audit Crypto separately) |

### 17.2 Adversaires HORS scope

Explicitement non-résistants :

- **Global Passive Adversary** (NSA-level, 5-Eyes coalition). Avec
  visibilité sur 60%+ du trafic mondial, la corrélation timing
  brise tous les mixnets connus à latence < 1h. Gotham vise <300 ms,
  donc vulnérable.
- **Compromission majoritaire des relais** (>60% du pool hostile).
  Si l'attaquant contrôle entry + exit de la majorité des chemins,
  il peut corréler par timing même avec mixing.
- **Coercition jurdictionnelle complète d'un opérateur** + **majeur
  pourcentage des relais opérés par la même entité**. Si tous nos
  relais tournaient en France, une décision de juge français
  contraindrait l'opérateur entier. Mitigation : exigence multi-
  opérateurs multi-pays dans la directory policy.
- **Endpoint compromise**. Si l'OS de l'utilisateur est compromis,
  l'attaquant lit les messages avant chiffrement. Gotham ne peut
  rien contre ça.
- **Forced key disclosure non-éphémère**. Si l'attaquant force un
  utilisateur à dévoiler sa clé long-terme, les messages futurs sont
  lus. Les messages passés sont protégés par forward secrecy
  (X25519 éphémère par message dans sealed-sender).

### 17.3 Garanties cryptographiques

- **Sphinx unwrap correctness** : la fonction `unwrap_header` produit
  l'output attendu OU `Err(BadMac)` sur input adversarial. Couvert
  par fuzz `header_v2_decode` (libFuzzer, totalité bornée par
  cargo-fuzz coverage) + proof Kani `header_v2_decode_never_panics`.
- **Sealed-sender ciphertext indistinguability** : sans `recipient_sk`,
  un attaquant ne peut PAS distinguer deux envelopes pour le même
  recipient. Garanti par ChaCha20-Poly1305 IND-CCA2 + fresh nonce
  par envelope.
- **Forward secrecy** : compromise de la clé long-terme sender →
  messages passés non-déchiffrables (clé éphémère détruite). Garanti
  par `EphemeralSecret::random_from_rng` qui détruit le sk après le DH.

### 17.4 Limites connues

- **1 B fuit en v0.1** : le `hop_index` byte révèle la position dans
  le chemin (corrigé en v0.2).
- **Pas de PQ dans le wire v0.1** : le format paquet 384 B ne contient
  pas le ML-KEM ciphertext (1088 B trop volumineux). Hybride PQ vit
  dans le module `hybrid.rs` mais pas wired.
- **Pas de per-hop integrity sur le payload v0.1** : un middle hop
  peut tamper le payload, détecté seulement au dernier hop ou par le
  destinataire (sealed-sender AEAD). v0.2 ajoute AEAD payload au
  dernier hop pour détecter le tampering AVANT delivery.

---

## 18. Comparaison avec d'autres systèmes

| Système | Type | Latence | PQ | Cover traffic | Status |
|---|---|---|---|---|---|
| **Tor** | Onion-router | 800-2000 ms | | standard | Production-mature |
| **Mixminion** | Mixnet anonyme | 1-4 h | | par-batch | Discontinued |
| **Loopix** | Mixnet temps-réel | ~1 s | | Poisson | Académique (papier 2017) |
| **Nym** | Mixnet + Loopix | ~1-2 s | migration en cours | | Production (token-incentivized) |
| **HOPR** | Mixnet on-chain | quelques s | | | Token-incentivized |
| **Signal** | Messagerie E2E | ~100 ms | | | Production-mature |
| **Wire** | Messagerie E2E | ~100 ms | | | Production |
| **Olvid** | Messagerie E2E | ~100 ms | | | Production FR |
| **Tchap (Matrix)** | Messagerie fédérée | ~100 ms | | | Production État FR |
| **Gotham** | Mixnet temps-réel | 50-300 ms | ML-KEM-768 hybride | Poisson Loopix | Pre-alpha, en build |

### 18.1 Détails vs Tor

Tor est le standard de facto pour l'anonymat réseau. Gotham emprunte
beaucoup à Tor (3 hops, onion encryption, directory authority) mais
diffère sur :
- Pas de circuits long-vivants (chaque paquet = nouveau chemin).
- Pas de SOCKS proxy général-purpose (dédié messagerie).
- Format Sphinx vs format Tor cell.
- Mixing Poisson vs forwarding immédiat.
- PQ hybride.

### 18.2 Détails vs Nym

Nym est le concurrent le plus proche techniquement. Différences :
- Nym = token NYM blockchain pour incentiviser les operators. Gotham
  = open + commercial (relais opérés par opérateurs sous accord).
- Nym = Sphinx classique. Gotham = Sphinx + PQ hybride v0.2.
- Nym = anonymity-as-a-service multi-applications. Gotham = couche
  transport dédiée pour Crypto + autres apps souveraines.

### 18.3 Détails vs Signal

Signal = messagerie E2E avec sealed-sender. Gotham = transport
anonyme orthogonal. **Les deux sont complémentaires** : Crypto utilise
le ratchet Signal-style (X3DH + Double Ratchet) pour le contenu, et
Gotham pour le transport métadonnée-cachant. Le sealed-sender envelope
de Crypto est inspiré directement de Signal.

---

## 19. Hardening + assurance

### 19.1 Lint discipline

- `#![cfg_attr(not(test), deny(clippy::unwrap_used))]`
- `#![cfg_attr(not(test), deny(clippy::expect_used))]`
- `#![warn(missing_docs)]`
- `cargo clippy --workspace --all-targets` : **0 warnings**.
- `cargo test --workspace` : **206+ tests, 0 failed**.

### 19.2 Fuzzing (cargo-fuzz)

5 cibles dans `crypto-gotham/fuzz/fuzz_targets/` :
- `header_decode` — v0.1 Header::decode totalité sur 384 B.
- `header_v2_decode` — v0.2 decode + tampered-unwrap (BadMac dominance).
- `sealed_unseal` — sealed envelope parser sur input adversarial.
- `directory_refresh_decode` — JSON decoders.
- `payload_v2_unframe` — length-prefix framing.

Run cycle : 60 s par cible en CI loop, overnight focused fuzz pour
les changes touchant la couche crypto.

### 19.3 Kani model-checking

6 preuves dans `crypto-gotham/proofs/lib.rs` :
1. `header_v1_decode_never_panics` — totalité décodeur v0.1.
2. `header_v2_decode_never_panics` — totalité décodeur v0.2.
3. `header_v2_rejects_bad_version` — ANY byte ≠ 2 → Err.
4. `header_v2_rejects_nonzero_reserved` — reserved bytes ≠ 0 → Err.
5. `unframe_plaintext_total_on_small_inputs` — totalité sur 32 B.
6. `dir_req_decode_total_small` — totalité sur 16 B.

Bounded-input proofs (5, 6) couvrent la surface parseur ; full input-
space couverte par libFuzzer. Combinaison = couverture quasi-formelle
à coût tractable.

### 19.4 Dudect timing-leak benchmarks

`crypto-gotham/benches/timing_leak.rs` — Criterion benches deux-classes
A/B (valide vs adversarial) pour :
- `header_v2::unwrap_header` (BadMac early-return path).
- `sealed::unseal` (AEAD failure path).

Distribution overlapping → no detectable timing leak.
Distribution bimodal → P1 investigation (replace `==` par
`subtle::ConstantTimeEq`, audit early-return branches, etc.).

### 19.5 Audit externe (roadmap)

Engagement prévu :
- **Quarkslab** (FR) ou **Synacktiv** (FR) ou **NCC Group** (UK/US)
  pour le crate `crypto-gotham` core.
- Coût ~50-80 k€ pour audit + remédiation Phase 1.
- Gate sur le wiring v0.2 — pas de promotion en default tant que
  l'audit n'a pas signé clean.

---

## 20. Déploiement opérationnel

### 20.1 Bootstrap d'un relais

Préparation host :
1. VPS dédié, Debian 12+ minimal (ou Arch hardened).
2. Compte unprivileged `gotham`.
3. UDP/443 ouvert (et TCP/443 quand pluggable v0.2 actif).
4. Pas de SSH passwd-auth (clé only), `fail2ban` actif.

Install :
```bash
# Build le binaire (host dev)
cargo build --release -p crypto-gotham-relay

# Copy vers le VPS
scp target/release/gotham-relay vps:/usr/local/bin/

# Générer la clé X25519 long-terme (offline)
head -c 32 /dev/urandom > /etc/gotham-relay/static.key
chmod 0600 /etc/gotham-relay/static.key

# Install service systemd
cp deploy/gotham-relay.service /etc/systemd/system/
systemctl enable --now gotham-relay
```

### 20.2 Enrollment dans le directory

L'opérateur soumet à l'authority un descripteur :
```json
{
  "id_pubkey_hex": "...",
  "kem_pubkey_hex": "...",
  "addr": "1.2.3.4:443",
  "tier": "Mix",
  "country": "FR",
  "asn": 12345,
  "operator": "Operator-Alpha",
  "uptime_pct": 99.5
}
```

Authority valide (DNS reverse-lookup, port reachability, key
fingerprint match), ajoute à la doc, signe, publie. Renewal quotidien.

### 20.3 Monitoring (privacy-preserving)

`/metrics` endpoint Prometheus exposé sur localhost uniquement (pas
public). Pull via `node_exporter` ou direct depuis Prometheus.

Alertes recommandées :
- `gotham_relay_uptime_seconds < 300` (juste restarté → investigate)
- `gotham_relay_packets_dropped_total{reason="bad_mac"} / total > 10%`
  (anomalie réseau ou attaque)
- `gotham_relay_pool_size = 0` (peer connectivity issue)

### 20.4 Key rotation

`static_sk` du relais : rotation tous les **30 jours**.

Procédure :
1. Génère nouvelle clé `gotham-relay-keygen > new.key`.
2. Authority signe une nouvelle directory annonçant la nouvelle
   clé (transition window 24h pendant lequel les deux clés sont
   valides).
3. SIGHUP au relais → reload la nouvelle clé.
4. Après 24h, retire l'ancienne clé.

v0.2 : automated rotation via Tang / Clevis pour TPM-backed unseal.

### 20.5 Network topology recommandée

Production minimum :
- **5 relais** minimum (1 Entry, 3 Mix, 1 Exit).
- **3 pays UE distincts** (FR, DE, IT par exemple) — coercition
  juridictionnelle multi-pays nécessaire pour briser l'anonymat.
- **3 opérateurs distincts** — collusion difficile.
- **Capacité bandwidth** : 100 Mbps minimum par relais en hot path.

Cible long-terme : 20-50 relais distribués, opérés par un mix
d'opérateurs commerciaux (Tier-1 cyberseccurity) + opérateurs
communautaires (universités, ONG techniques).

---

## 21. Roadmap technique

### 21.1 Court-terme (3 mois)

| Item | Statut |
|---|---|
| Audit externe Quarkslab/Synacktiv lancé | À planifier |
| Wiring v0.2 (`header_v2` + `payload_v2`) dans `packet.rs` / `process.rs` | Gate sur audit |
| 5 relais publics multi-pays déployés | À planifier |
| Authority Ed25519 sur YubiKey | À planifier |
| CSPN ANSSI dossier déposé | À préparer |

### 21.2 Mid-terme (6-12 mois)

| Item | Détails |
|---|---|
| **A6.3 obfs4 transport** | Intégration `obfs4-rs` ou port clean-room |
| **A6.4 meek-CDN transport** | Cloudfront / Fastly account + protocol |
| **A6.6 detection-evasion harness** | nDPI + Suricata test fixtures en CI |
| **A8 APNS hot path** | a2 + ES256 JWT signing |
| **A8 FCM hot path** | reqwest + OAuth2 service-account |
| **A8 Admin enrollment endpoint** | HTTP localhost-only |
| **Per-hop payload AEAD complet** | Upgrade vers LIONESS (v0.3) |
| **Model B subscribe protocol** | Pour users derrière NAT (mobile-first) |
| **Invitation URI extends `gotham_pk`** | Auto key-exchange (élimine paste manuel) |

### 21.3 Long-terme (12-24 mois)

| Item | Détails |
|---|---|
| **CSPN obtenue + Visa de sécurité ANSSI** | 12 mois calendaire, 80-120 k€ |
| **SecNumCloud** (si SaaS hosted offer) | 12-18 mois, 100-200 k€ |
| **Multi-sig directory N-of-M** | Plusieurs authorities, élimine SPoF |
| **A7 formel Kani complet** | Couverture exhaustive (vs scaffold actuel) |
| **A7 dudect t-test CI gating** | Pipeline automatisé timing leak detection |
| **Bug bounty program HackerOne** | 5 k€ pot minimum |
| **CVE Numbering Authority** | Registration |
| **IETF informational track RFC** | Soumission pour crédibilité |
| **Hardware TRNG support** | Pour relais opérateurs production |
| **Air-gapped key ceremony** | Procédure formelle pour authority key |

---

## 22. Glossaire

| Terme | Définition |
|---|---|
| **AEAD** | Authenticated Encryption with Associated Data. Chiffrement + intégrité en un appel. |
| **α (alpha)** | Clé publique X25519 éphémère portée dans le header Gotham, re-blindée par-hop. |
| **β (beta)** | Routing block dans le header — encrypted routes per-hop. |
| **γ (gamma)** | MAC Poly1305 authentifiant le header. |
| **AGPLv3** | GNU Affero General Public License v3 — licence open-source viral pour code utilisé en service réseau. |
| **APNS** | Apple Push Notification Service. |
| **AS** | Autonomous System (BGP). /16 = bloc IP de 65536 adresses. |
| **CSPN** | Certification de Sécurité de Premier Niveau (ANSSI). |
| **DPI** | Deep Packet Inspection — analyse profonde de paquets pour bloquer/identifier des protocoles. |
| **EdDSA** | Edwards-curve Digital Signature Algorithm (Ed25519 = variante Curve25519). |
| **FCM** | Firebase Cloud Messaging. |
| **FIPS** | Federal Information Processing Standards (NIST). |
| **GPA** | Global Passive Adversary — attaquant avec visibilité réseau globale. |
| **HKDF** | HMAC-based Key Derivation Function. |
| **KEM** | Key Encapsulation Mechanism — primitive pour échanger des secrets. |
| **LRU** | Least Recently Used (cache). |
| **MAC** | Message Authentication Code (intégrité + authenticité). |
| **ML-KEM** | Module-Lattice KEM (Kyber rebranded, FIPS 203). |
| **MTU** | Maximum Transmission Unit — taille max d'un paquet IP. |
| **NAT** | Network Address Translation. |
| **Noise XK** | Pattern Noise Protocol où le client connaît la pubkey serveur (eXtended Knowledge). |
| **OIDC** | OpenID Connect — auth standard sur OAuth2. |
| **PKCE** | Proof Key for Code Exchange — extension OAuth2 anti-CSRF. |
| **PQ** | Post-quantique (résistant aux ordinateurs quantiques). |
| **QUIC** | Quick UDP Internet Connections — transport sur UDP avec TLS intégré. |
| **RGPD** | Règlement Général sur la Protection des Données (EU GDPR). |
| **RUSTSEC** | Database d'advisories pour l'écosystème Rust. |
| **SCIM** | System for Cross-domain Identity Management — provisioning automated. |
| **SecNumCloud** | Référentiel ANSSI pour services cloud français. |
| **SNI** | Server Name Indication — extension TLS exposant le hostname. |
| **Sphinx** | Format paquet onion-routing (Danezis-Goldberg 2009). |
| **SOCKS5** | Protocol proxy générique. |
| **SQLCipher** | Extension SQLite chiffrant la database. |
| **SURB** | Single-Use Reply Block — pre-built reply route pour mixnet anonyme. |
| **TLS** | Transport Layer Security (1.3 = standard 2018). |
| **TPM** | Trusted Platform Module. |
| **TTL** | Time To Live. |
| **X25519** | ECDH sur Curve25519 (RFC 7748). |
| **X3DH** | Extended Triple Diffie-Hellman — protocole établissement session Signal. |

---

## 23. Références

### Standards

- **RFC 7748** — Curve25519, Curve448 (X25519/X448).
- **RFC 8032** — Ed25519, Ed448 (EdDSA).
- **RFC 5869** — HKDF.
- **RFC 8439** — ChaCha20 + Poly1305 (ChaCha20-Poly1305 AEAD).
- **RFC 9106** — Argon2 (Argon2id paramters).
- **NIST FIPS 203** — ML-KEM (Kyber Module-Lattice KEM standard).
- **NIST FIPS 204** — ML-DSA (Dilithium signatures).
- **NIST FIPS 205** — SLH-DSA (Sphincs+ signatures).

### Articles académiques

- Chaum, D. (1981). *Untraceable Electronic Mail, Return Addresses,
  and Digital Pseudonyms*. **CACM 24(2)**.
- Danezis, G., Goldberg, I. (2009). *Sphinx: A Compact and Provably
  Secure Mix Format*. **IEEE S&P 2009**.
- Piotrowska, A. M. et al. (2017). *The Loopix Anonymity System*.
  **USENIX Security 2017**.
- Dingledine, R., Mathewson, N., Syverson, P. (2004). *Tor: The Second-
  Generation Onion Router*. **USENIX Security 2004**.
- Cohn-Gordon, K., Cremers, C., Garratt, L., Millican, J., Milner, K.
  (2018). *On Ends-to-Ends Encryption: Asynchronous Group Messaging
  with Strong Security Guarantees*. **CCS 2018**.
- Marlinspike, M., Perrin, T. (2016). *The X3DH Key Agreement Protocol*.

### Tools & bibliothèques utilisés

- `x25519-dalek 2.x` — X25519 pure Rust.
- `ed25519-dalek 2.x` — Ed25519 pure Rust.
- `ml-kem 0.2` — ML-KEM-768 pure Rust.
- `chacha20poly1305 0.10` — AEAD ChaCha20-Poly1305.
- `poly1305 0.8` — Poly1305 MAC.
- `hkdf 0.12` — HKDF.
- `sha2 0.10` — SHA-256.
- `blake3 1.x` — BLAKE3.
- `snow 0.9` — Noise Protocol implementation Rust.
- `quinn 0.11` — QUIC implementation Rust.
- `rustls 0.23` — TLS 1.3 implementation Rust.
- `tokio 1.x` — async runtime.
- `serde / serde_json` — sérialisation.
- `subtle` — constant-time comparisons.
- `zeroize` — wipe secrets in memory.

### Documents internes du projet

- `GOTHAM.md` — spécification protocole.
- `GOTHAM-CHECKLIST.md` — roadmap d'implémentation.
- `SECURITY.md` — advisories + mitigations.
- `PITCH.md` — pitch investisseur.
- `crypto-gotham/HARDENING.md` — runbook fuzz + Kani + dudect.
- `crypto-gotham/LICENSE-AGPL` — texte AGPLv3.
- `crypto-gotham/LICENSE-COMMERCIAL` — texte licence commerciale.

---

*« L'eau persiste là où le Fremen la cache. L'épice afflue où le
travail est fait. »*
