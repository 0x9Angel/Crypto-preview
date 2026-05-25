# SECURITY AUDIT — Crypto + Gotham

**Date** : 2026-05-25
**Auditeur** : Angel (interne)
**Périmètre** : `crypto/` workspace complet — 27 189 LOC Rust + 7 103 LOC TS/React, 765 crate dependencies.
**Méthode** : revue manuelle ligne-par-ligne + `cargo audit` + recherche de patterns dangereux (SQLi,
unsafe, RNG, nonce reuse, panic paths, XSS, path traversal).

## TL;DR

- **0 vulnérabilité critique** (RCE, contournement d'auth, récupération de clé)
- **4 vulnérabilités HIGH** : 3 par dépendance tierce + 1 DoS via `total_chunks`
- **8 vulnérabilités MEDIUM** dont 1 protocolaire (Sphinx low-order points)
- **5 LOW** (qualité défense en profondeur)
- **23 dépendances unmaintained** (gtk-rs surtout, transitif via Tauri)

L'application est globalement bien construite : primitives crypto correctement utilisées,
zeroization sur les types secrets, sanitisation des entrées peer-supplied, throttle anti-brute-force
sur unlock, CSP stricte côté webview, IPC commands typés. Les findings concernent essentiellement
la défense en profondeur et le hardening contre les attaques DoS.

---

## HIGH

### H-1 — DoS via `total_chunks` non borné dans les transferts de fichiers
**Fichier** : [crypto-store/src/crud.rs:1643-1659](crypto-store/src/crud.rs#L1643-L1659)
`init_file_transfer` accepte `total_chunks: u32` directement du peer. Un attaquant envoie
`total_chunks = 4_294_967_295` (u32::MAX). Conséquences :
- Le `(0..total).filter(...)` de `missing_file_chunks` alloue ~4 milliards d'itérations → OOM
- `complete_file_transfer` ne se termine jamais (chunks_received < total)
- `gc_orphan_file_chunks` finit par nettoyer (24 h) mais entre-temps DoS confirmé

**Patch** : cap à 50 000 chunks (3.2 GiB max si chunk = 64 KiB), valider `chunk_index < total_chunks`
dans `save_file_chunk`. Patché en local (commit suivant).

### H-2 — Pas de cap sur les connexions concurrentes du relais
**Fichier** : [crypto-server/src/server.rs:125-145](crypto-server/src/server.rs#L125-L145)
La boucle `loop { listener.accept() ... tokio::spawn(...) }` ne borne ni le nombre de connexions
ouvertes simultanément ni le rythme d'acceptation par IP. Le rate limiter ne tape qu'**après** le
handshake Noise complet — un attaquant peut donc forcer le serveur à effectuer une opération DH
X25519 + symmetric crypto par tentative de connexion sans coût.

**Patch** : (a) `Semaphore` global limitant à N (par défaut 10 000) connexions concurrentes ; (b) per-IP
connect-rate-limit avant le handshake. Patché en local.

### H-3 — RUSTSEC-2026-0098 / 0099 / 0104 (rustls-webpki 0.101.7) FIXÉ
**Statut** : Patché par upgrade `openidconnect 3.5 → 4.0.1`. La nouvelle chaîne tire
`oauth2 5.0 + reqwest 0.12 + rustls 0.23 + rustls-webpki 0.103`. Les trois CVE rustls-webpki
sont éliminés.

### H-4 — RUSTSEC-2023-0071 (rsa 0.9.10 — Marvin Attack) PERSISTANT
**Statut après upgrade openidconnect 4** : `rsa 0.9.10` est désormais une dépendance directe
d'`openidconnect 4.0.1` lui-même (utilisée pour le déchiffrement de JWT chiffrés en RSA-OAEP /
RSA-PKCS1). **Aucune version corrigée disponible** chez RustCrypto. Mitigations en place :
- SSO uniquement en tier Enterprise (utilisateurs Community non exposés)
- L'attaquant doit être en MitM réseau prolongé pour exploiter le canal de timing
- Les IdP majeurs (Google, Microsoft, Okta) signent en RSA-PSS, pas RSA-PKCS1
- À tracker : RustCrypto/RSA-PR#394 (constant-time RSA en cours de revue)

---

## MEDIUM

### M-1 — Sphinx `blind_alpha` n'écarte pas les points X25519 de petit ordre
**Fichier** : [crypto-gotham/src/header.rs:350-352](crypto-gotham/src/header.rs#L350-L352)
`x25519(*k_blind, *alpha)` n'inspecte pas si `alpha` est un des 8 points de petit ordre de la
courbe Montgomery. Un relais en amont malveillant pourrait substituer α par un tel point ;
tous les relais en aval dériveraient alors le même secret partagé prédictible (0 ou
constante connue).

**Statut** : limitation documentée v0.1, le v0.2 doit corriger. La défense actuelle est
implicite : l'ephemerals master scalar du sender est zéroïsé après usage, donc seul un relais
*malveillant en chemin* peut tenter l'attaque.

### M-2 — Eviction non-déterministe des `skipped_keys` dans le Double Ratchet
**Fichier** : [crypto-agent/src/ratchet.rs:232-236](crypto-agent/src/ratchet.rs#L232-L236)
Quand le cache atteint `MAX_TOTAL_SKIPPED_KEYS = 1000`, `skipped_keys.keys().next().copied()`
évacue une entrée arbitraire (l'ordre des clés dans un `HashMap` est aléatoire). Conséquences :
- Un attaquant qui force le pair à recevoir 1000+ messages out-of-order peut éjecter
  des clés de messages légitimes futurs
- Aucune garantie LRU / FIFO

**Patch** : utiliser `IndexMap` pour avoir une ordre d'insertion stable + éviction FIFO. Patché.

### M-3 — `skipped_keys` n'est pas zéroïsé au drop
**Fichier** : [crypto-agent/src/ratchet.rs:36-37](crypto-agent/src/ratchet.rs#L36-L37)
Le champ `skipped_keys: HashMap<([u8;32], u32), [u8;32]>` porte `#[zeroize(skip)]`. Les clés de
messages skipped restent en mémoire libérée jusqu'à réécriture. Combiné avec un swap disque, fuit
des clés AES-256-GCM utilisables pour déchiffrer hors-bande.

**Patch** : implémenter `Drop` manuel qui zéroïse chaque entrée avant de drop le HashMap. Patché.

### M-4 — License `verify` permet la malléabilité Ed25519
**Fichier** : [crypto-enterprise/src/license.rs:143](crypto-enterprise/src/license.rs#L143)
`pubkey.verify(...)` (au lieu de `verify_strict`) accepte les signatures non-canonique. Un
attaquant ayant un token valide peut en générer un autre avec une signature équivalente — sans
forger un nouveau payload, donc l'attaque n'a pas d'impact pratique sur la licence. Mais
`verify_strict` est la bonne défense en profondeur.

**Patch** : passer à `verify_strict`. Patché.

### M-5 — Mot de passe stocké en state React pour le flow B2
**Fichier** : [crypto-tauri/src/App.tsx:4341-4361](crypto-tauri/src/App.tsx#L4341-L4361)
`recoveryPwd` retient le mot de passe en clair dans le `useState` du composant racine jusqu'à
ce que `has_recovery` retourne `true` (premier launch d'un nouveau profil) ou que l'utilisateur
ferme la modale de setup. Fenêtre d'exposition : ~100 ms typique, jusqu'à plusieurs minutes
si l'utilisateur ignore la modale.

**Mitigation seulement** : un XSS exploiterait cette state. Avec la CSP stricte actuelle
(`script-src 'self'`), pas d'XSS classique. Mais en cas de bug WebKit (CVE) ce vecteur devient
exploitable.

**Patch proposé** (non appliqué — change l'UX) : ré-prompter le mot de passe juste avant la
seal du recovery envelope, plutôt que de le retenir.

### M-6 — CSP : `unsafe-inline` pour les styles + absence de directives de hardening
**Fichier** : [crypto-tauri/src-tauri/tauri.conf.json:25](crypto-tauri/src-tauri/tauri.conf.json#L25)
- `style-src 'self' 'unsafe-inline'` est imposé par Tailwind v4 (génération CSS-in-JS).
- Manque `object-src 'none'`, `frame-ancestors 'none'`, `base-uri 'self'`, `form-action 'none'`.

**Patch** : ajouter les directives manquantes. Patché.

### M-7 — Rate limiter `DashMap` non borné en taille
**Fichier** : [crypto-server/src/rate_limit.rs:51](crypto-server/src/rate_limit.rs#L51)
Un attaquant utilise une plage IPv6 large (2^64 adresses dans un /64) pour gonfler le `DashMap`
jusqu'au CLEANUP_INTERVAL (5 min). Au rythme MAX_COMMANDS_PER_SEC * cleanup_interval entrées
possibles, RAM peut grimper de plusieurs Go.

**Patch** : cap à 100 000 entrées avec éviction des plus anciennes. Patché.

### M-8 — Logs SCIM divulguent les `user_name` (souvent emails)
**Fichier** : [crypto-server/src/scim_server.rs:138](crypto-server/src/scim_server.rs#L138)
`info!(user_name = %user.user_name, ...)` écrit l'email PII en clair dans les logs. Combiné
avec une fuite de log (journald → backup non chiffré), c'est un mini-RGPD-incident.

**Patch** : hasher l'user_name dans les logs comme `redact_contact()` fait côté client. Patché.

---

## LOW

### L-1 — `RECOVERY_INFO` constante morte
**Fichier** : [crypto-store/src/recovery.rs:60](crypto-store/src/recovery.rs#L60)
Définie mais jamais utilisée dans la dérivation HKDF (le code utilise `argon2` directement).
Le commentaire suggérait une intention de domain-separation non implémentée. Pas d'impact
fonctionnel ; à nettoyer.

**Patch** : supprimer la constante. Patché.

### L-2 — `x3dh_respond_with_inv_key` ne vérifie pas la signature SPK
**Fichier** : [crypto-agent/src/x3dh.rs:312-339](crypto-agent/src/x3dh.rs#L312-L339)
Par design — la fonction utilise une clé d'invitation éphémère per-link, fournie hors-bande
via l'URI d'invitation. Le doc-comment ne l'explicite pas — quiconque relit le code pourrait
penser que c'est un bug. **À documenter, pas à patcher.**

### L-3 — 23 dépendances unmaintained
Principalement la chaîne gtk-rs (atk, gdk, gtk, etc.) transitive via Tauri 2 sur Linux. Pas de
CVE active, mais accumulation de tech debt. À suivre via cargo audit régulier.

### L-4 — `KDF V1` (HMAC-SHA256 avec clé hardcodée) accepté pour les bases legacy
**Fichier** : [crypto-store/src/kdf.rs:29-38](crypto-store/src/kdf.rs#L29-L38)
Une base créée avant l'introduction d'Argon2id (v1) est dérivable trivialement (HMAC d'un
mot de passe avec une clé constante connue). Le code la migre automatiquement vers v2 au
premier `open_database`. **Limite** : l'utilisateur qui n'ouvre jamais son app reste sur v1.
**Recommandation** : prévoir un script de migration forcée si jamais une compromission de
clé v1 est suspectée.

### L-5 — `unwrap()` / `panic!` en chemin chaud (811 occurrences hors tests)
Beaucoup proviennent de mutex `.lock().unwrap()` (poison-on-panic, acceptable) et de `expect()`
sur des invariants vérifiés statiquement. Aucun cas reviewé n'est exploitable, mais
une revue ciblée des chemins atteignables par l'IPC reste recommandée.

---

## INFO

- **Aucun `unsafe` dans le code applicatif** (un seul, dans un test de `crypto-enterprise/src/license.rs` pour mutation byte-by-byte).
- **Aucun pattern SQLi** détecté (toutes les requêtes utilisent `params!` / `?`).
- **Aucun shell exec** (`Command::new`, `std::process::Command`) hors paths déjà audités.
- **Aucune utilisation de RNG non-cryptographique** dans le code de production (toutes les
  occurences de `thread_rng()` sont en code de test ou pour de la sélection non-sensible).
- **Tauri capabilities** : minimal (`opener`, `notification`, `updater`, `dialog`). Pas de
  `plugin-fs` exposé au webview.
- **CSP `script-src 'self'`** : strict, pas d'`unsafe-eval`, pas d'`unsafe-inline` pour les scripts.
- **Pas de markdown unsafe** : le `renderMarkdown` custom utilise des nœuds React typés,
  pas de `dangerouslySetInnerHTML`. 

---

## Patches appliqués sans confirmation (sûrs, n'altèrent pas les protocoles)

| ID | Description | Fichier |
|---|---|---|
| H-1 | Cap `total_chunks` ≤ 50 000, valider `chunk_index < total_chunks` | crypto-store/src/crud.rs |
| H-2 | Semaphore + per-IP accept rate-limit | crypto-server/src/server.rs |
| M-2 | `IndexMap` FIFO pour `skipped_keys` | crypto-agent/src/ratchet.rs |
| M-3 | `Drop` manuel qui zéroïse `skipped_keys` | crypto-agent/src/ratchet.rs |
| M-4 | `verify_strict` au lieu de `verify` | crypto-enterprise/src/license.rs |
| M-6 | CSP hardening | crypto-tauri/src-tauri/tauri.conf.json |
| M-7 | Cap `DashMap` du rate limiter | crypto-server/src/rate_limit.rs |
| M-8 | Hash des user_name dans les logs SCIM | crypto-server/src/scim_server.rs |
| L-1 | Suppression `RECOVERY_INFO` | crypto-store/src/recovery.rs |

## Décisions à prendre (changent le protocole ou affectent l'UX/API)

| ID | Description | Risque si non-traité |
|---|---|---|
| H-3 | Upgrade `openidconnect` chain → `rustls-webpki ≥ 0.103.12` | MitM via cert wildcard / panic CRL |
| H-4 | RSA Marvin timing (pas de fix amont) | Récup. de clé SSO en MitM réseau prolongé |
| M-1 | Rejet des points X25519 de petit ordre dans Sphinx blind | Compromission d'anonymat partielle |
| M-5 | Refactor du flow B2 pour ne pas retenir le mot de passe en JS | Exfiltration via XSS hypothétique |
| L-4 | Migration forcée des bases v1 | Theoretical (utilisateurs offline depuis longtemps) |

---

## Recommandations opérationnelles

1. **Activer `cargo audit` dans la CI** (workflow GitHub Actions) — fail-on-error pour HIGH+, warn-only pour MEDIUM-.
2. **Pin les versions critiques** dans `Cargo.lock` — déjà fait, garder ainsi.
3. **Tester un fuzzer** sur les parsers (Sphinx header, X3DH init header, recovery envelope).
4. **Refaire un cycle audit complet avant la GA** : ce rapport couvre l'état au 2026-05-25 ; tout
   nouveau code (B10 multi-device, Track A v0.2) doit être audité avant merge.
5. **Pentest externe** une fois la levée seed bouclée (budget recommandé : 30-50 k€ pour 2 semaines).
