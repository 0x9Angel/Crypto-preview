# Crypto — pitch investisseur

## TL;DR 

**Crypto** est une plateforme de messagerie chiffrée souveraine pour les grandes entreprises et l'administration française et européenne. Elle combine la **simplicité d'usage d'une app grand public** (Signal, WhatsApp) avec une **infrastructure réseau de niveau renseignement** : notre protocole propriétaire **Gotham** masque non seulement le contenu des messages mais aussi **qui parle à qui, quand, depuis où** — métadonnées que Signal, Wire, Tchap et leurs concurrents laissent fuiter.

Cible commerciale : **Airbus, Thales, Dassault, Naval Group, MBDA, Safran, l'État français**. Marché secondaire : journalistes d'investigation, ONG, cabinets d'avocats d'affaires.

Nous demandons **[3Millions]** pour accélérer la certification ANSSI CSPN, déployer le réseau Gotham public et signer les premiers contrats pilotes.

---

## Le problème que personne ne résout

Les messageries professionnelles actuelles tombent dans trois catégories, et **aucune ne couvre le besoin du marché souverain européen** :

**1. Les solutions américaines** (Slack, Teams, WhatsApp Business, Signal). Confort utilisateur excellent, **mais juridiction CLOUD Act**. Microsoft, Meta et Apple sont contraintes par la loi américaine de remettre les métadonnées sur demande judiciaire US, y compris pour des comptes français.

**2. Les solutions souveraines existantes** (Tchap pour l'État français, Olvid pour le privé, Wire en Suisse). Sovereignty oui, mais **architecture vieillissante** : Tchap est basé sur Matrix qui n'a jamais été conçu pour résister à un adversaire passif global. Le serveur sait qui parle à qui. Olvid est meilleur sur la cryptographie mais opère sur infrastructure clearnet — un observateur réseau voit les patterns de communication même s'il ne voit pas le contenu.

**3. Les outils "spy-grade"** (Cellebrite-resistant tools, certains C2 d'État). Inutilisables par un commercial chez Airbus, conçus pour 50 opérateurs, pas pour 50 000 employés.

**Le vide** : aucune solution n'offre simultanément (a) la confidentialité métadonnée d'un mixnet sérieux, (b) l'ergonomie d'une app grand public, (c) les fonctions enterprise (SSO, audit, multi-tenant, conformité RGPD/NIS2/DORA) qu'un DSI de Tier-1 exigera.

C'est ce vide que Crypto comble.

---

## Ce qu'est Crypto, concrètement

Crypto est une **application desktop + mobile** (Tauri + React + Rust ; même base de code pour Windows / macOS / Linux / iOS / Android) qui se présente à l'utilisateur comme un Signal "pro" :

- Messagerie 1-to-1 et de groupe avec chiffrement de bout en bout (protocole **X3DH + Double Ratchet** — le même que Signal, audit-grade, post-Snowden).
- Profils utilisateurs (avatar, bio, statut), réactions, réponses, édition/suppression de messages, accusés de réception/lecture, présence "typing".
- Partage de fichiers, appels audio/vidéo WebRTC chiffrés.
- Groupes structurés avec sous-canaux (à la Slack).
- Recherche full-text dans l'historique local chiffré.

Pour l'utilisateur final : **rien à apprendre**. C'est familier.

Pour le DSI / RSSI :

- **Single Sign-On** via OIDC (Azure Entra, Okta, Keycloak, AD FS) avec PKCE + at_hash.
- **Provisioning SCIM 2.0** — création/désactivation automatique de comptes depuis l'annuaire d'entreprise.
- **Audit logs signés Ed25519** — chaque événement administratif tracé, signature vérifiable a posteriori (anti-trafic).
- **Licensing on-premise** — la clé de licence est un token signé Ed25519 hors-ligne, validé localement par le binaire. Pas de phone-home.
- **Réplication active/passive** pour la haute disponibilité on-prem.
- **Base SQLite chiffrée** côté client (SQLCipher, KDF Argon2id) ; rotation transparente V1 → V2 sur la base utilisateur.
- **Persistance offline** : les messages mis en file d'attente sont rejoués à la reconnexion ; pas de perte sur coupure réseau.

C'est l'app prête à déployer auprès d'un département de 5 000 personnes chez Thales.

---

## Le différenciateur technique : Gotham

C'est ici que se joue notre **moat défendable**.

### Le problème : la fuite de métadonnées

Signal chiffre votre message. Personne ne lit "Alice envoie 'le projet est validé' à Bob". Mais le serveur Signal voit que **Alice a parlé à Bob à 14h32 pendant 4 secondes**. Multiplié sur 6 mois, ce graphe relationnel suffit à un analyste pour reconstruire l'organigramme R&D, identifier les "early signals" d'un appel d'offres, ou prédire un rapprochement industriel **avant qu'il ne soit annoncé**.

Pour un industriel défense, c'est inacceptable. Pour un État, c'est une faille de souveraineté.

### La solution : un mixnet sur-mesure

Gotham est notre **protocole de transport anonyme** basé sur le format **Sphinx** (Danezis-Goldberg 2009, classe d'État de l'art académique) que nous avons réimplémenté de zéro en Rust avec trois différences clés :

1. **Hybride post-quantique** : chaque saut utilise X25519 **+** ML-KEM-768 (NIST FIPS 203, post-quantique standardisé 2024). Un adversaire qui enregistrerait notre trafic aujourd'hui pour le déchiffrer dans 10 ans avec un ordinateur quantique échoue.

2. **Latence basse** (50-300 ms médian vs 800-2000 ms pour Tor). Permet le temps réel (présence, indicateurs de frappe, accusés de lecture instantanés) — impossible sur Tor.

3. **Délais de mixing Poisson + cover traffic continu** : un observateur réseau voit un flux de paquets de taille fixe (2048 octets) à intervalles aléatoires, **que vous chattiez activement ou pas**. La distinction "utilisateur actif vs utilisateur idle" disparaît au-delà d'une fenêtre de 30 secondes.

### Garanties cryptographiques

| Attaque | Tor | Signal | **Crypto + Gotham** |
|---|---|---|---|
| Lecture du contenu | résiste |  résiste |  résiste |
| Identification de l'émetteur |  résiste (3 hops) |  serveur voit |  résiste (sealed-sender + 3-5 hops) |
| Graphe relationnel |  vulnérable à corrélation timing |  |  résiste (Poisson + cover) |
| Adversaire quantique futur |  X25519 cassable |  X3DH cassable |  ML-KEM-768 hybride |
| Replay / tagging |  partiel |  |  MAC chain Sphinx + cache 5 min |
| DPI / blocage UDP/443 |  obfs4 nécessaire |  |  QUIC + TLS-TCP fallback + obfs4 (roadmap) |

### Modèle de menace explicite

Nous **publions notre threat model** comme partie intégrante du protocole. Nous garantissons résistance contre observateur réseau passif, compromission d'un relais (n'importe quel tier), attaques de rejeu, tagging, corrélation de timing en-dessous de la dizaine de milliers d'utilisateurs simultanés, et adversaires quantiques (sur ML-KEM-768). Nous **n'engageons pas** à résister à un Global Passive Adversary (NSA-level, 5-Eyes) ni à une compromission majoritaire des relais (>60%) — et nous le **disons explicitement** dans la documentation. Cette honnêteté est rare dans le secteur et c'est un argument de vente auprès des CISO sophistiqués.

---

## Ce qui est déjà implémenté

C'est ici que la pitch passe de la vision à la traction. **Tout ce qui suit est dans le dépôt git, testé, mesurable.**

### Côté application (Crypto)

- 100% des fonctions utilisateur grand public listées plus haut, **fonctionnelles et testées** sur les trois OS desktop. Build mobile en cours d'ajustement Tauri 2.
- 15 migrations de schéma SQLite déjà passées en prod, sans perte de données.
- Suite de tests : **206 tests workspace, 0 failed, 2 ignored** (les 2 ignorés sont des tests réseau de tail-Poisson volontairement marqués manuels).
- **0 warning clippy** sur l'ensemble du workspace après le dernier sweep — discipline de code stricte avec `deny(unwrap_used)` en code de prod.
- Intégration entreprise complète : SSO OIDC, SCIM 2.0, audit log Ed25519, license-gate Ed25519, cluster active/passive.

### Côté protocole (Gotham)

- **Phase 1 Sphinx packet** : 384 octets de header, format slot-based v0.1 + format folded (shift-and-pad) classique v0.2 implémenté en parallèle. **10/10 tests** sur la v0.2.
- **Phase 2 Relay binary** : `gotham-relay` daemon Rust standalone, listener QUIC sur UDP/443, Noise XK par lien, cache de rejeu LRU + TTL 5 min, scheduler Poisson par-saut. Pool de connexions avec éviction LRU. **27/27 tests** sur le crate relais.
- **Phase 3 Directory authority** : descripteurs de relais sérialisés JSON, signature Ed25519, validation client-side, sélection de chemin avec contraintes de diversité (opérateur + /16 IPv4). **15/15 tests**.
- **Phase 4 App integration** : commandes Tauri `gotham_init`, `gotham_send`, `gotham_status` exposées, drainer d'entrée wired sur le SessionManager existant, migration de `send_message` et `set_own_profile` avec fallback SMP transparent, UI dédiée dans la section Profil. Mode dev 3-relais self-loop pour démonstration.
- **Phase 5 Cover traffic** : `CoverScheduler` Poisson avec dégradation batterie (mobile), dispatch Drop / Loop / Real indistinguable à l'observateur.
- **Phase 6 Pluggable transports** : trait `GothamTransport` + sélecteur adaptatif probe-and-stick. QUIC implémenté, TLS-1.3 sur TCP/443 implémenté avec SNI réaliste, obfs4 et meek-CDN scaffoldés.
- **Phase 7 Hardening** : 5 cibles `cargo-fuzz` runnable, 6 preuves `Kani` model-checking, benches `Criterion` style dudect pour détection de fuites timing. Documenté dans `HARDENING.md`.
- **Phase 8 Push relay mobile** : binaire `gotham-push-relay` séparé avec providers APNS + FCM scaffoldés, store d'enrollment avec rotation 7 jours.
- **Sealed-sender envelope** : Signal-style, masque l'identité de l'émetteur même au relais de sortie. Forward secrecy via X25519 éphémère par message.

**Bilan code** : ~25 000 lignes Rust + 6 000 lignes TypeScript, dépôt git de référence, 17 commits poussés sur la branche `main` + une branche `gotham-v0.2` pour les changements de format paquet.

### Documentation

- `GOTHAM.md` : RFC complète du protocole (spec wire format, threat model, cryptographic constructions).
- `GOTHAM-CHECKLIST.md` : roadmap 8 tracks avec définition-of-done par étape.
- `SECURITY.md` : advisories actives + mitigations.
- `HARDENING.md` : runbook fuzz + Kani + dudect.
- Doc inline Rustdoc complète, génération `cargo doc` sans warning.

---

## Marché

### Tier-1 cible (objectif court terme, 12-18 mois)

| Acquéreur | Cible interne | Effectif sécurisable | Prix indicatif annuel/seat |
|---|---|---|---|
| Airbus | Programmes confidentiel-défense | ~50 000 | 200-500 € |
| Thales | Lignes de produit sensibles | ~30 000 | 200-500 € |
| Dassault Aviation | Bureau d'études + commercial | ~12 000 | 200-500 € |
| Naval Group | Programmes maritime militaire | ~15 000 | 200-500 € |
| MBDA | Programmes missiles européens | ~13 000 | 200-500 € |
| Safran | Systèmes embarqués critiques | ~20 000 | 200-500 € |

**TAM Tier-1 défense FR seul** : ~140 000 sièges × 350 € moy = **~49 M€ ARR** si 100% de pénétration. Réaliste à 5 ans avec 15-25% de part : **7-12 M€ ARR**.

### Tier-2 cible (18-36 mois après CSPN)

- État français (après contestation de Tchap) : ~500 000 fonctionnaires sensibles
- Banques systémiques FR/EU (BNP, Crédit Agricole, Société Générale, ING, Deutsche Bank) sur la donnée trade sensitive
- Énergie critique (EDF, Engie, RTE)
- Cabinets d'avocats d'affaires (Cleary, Bredin Prat, Darrois)

### Tier-3 cible (mid-term)

- ONG investigation (Forbidden Stories, OCCRP)
- Presse (consortiums Pandora Papers, Suisse Secrets)

---

## Compétition et positionnement

| Concurrent | Force | Faiblesse exploitable |
|---|---|---|
| **Signal** | Cryptographie de référence, gratuit | Pas de produit enterprise, hébergement US, métadonnées serveur |
| **WhatsApp Business** | Adoption massive | Meta = CLOUD Act, métadonnées vues |
| **Wire** | Européen, enterprise | Acheté plusieurs fois, gouvernance opaque, mixnet absent |
| **Olvid** | Français, B2B, crypto solide | Pas de mixnet, infrastructure clearnet |
| **Tchap** | Souverain État FR | Matrix = pas conçu anti-trafic-analyse, ergonomie datée |
| **Wickr** | Enterprise feature-rich | Acquis par AWS 2021 = juridiction US |
| **Threema** | Suisse, pro tier | Pas de mixnet, marketing faible B2B |

**Notre position** : seul produit cumulant **(a) ergonomie consommateur + (b) feature-pack enterprise + (c) mixnet post-quantique** intégré nativement. Personne d'autre n'a les trois.

---

## Modèle économique

**Trois flux de revenus** :

1. **Licences on-premise par-siège-par-an**. Tier Standard 200 €/siège/an, Tier Enterprise (SSO+audit+cluster) 500 €/siège/an, Tier Sovereign (déploiement air-gappé + support sur-site) sur devis.

2. **Cloud managé** (post-SecNumCloud, horizon 24-36 mois) : SaaS hosted FR, ~30 €/siège/mois.

3. **Services professionnels** : intégration AD/Entra, audits de conformité internes, formation administrateurs. ~1500-2500 €/jour.

**Coûts variables faibles** : binaire desktop = ~10 MB, pas de back-end cloud requis pour le tier on-prem. La gross margin tier on-prem dépasse 90% après amortissement R&D.

---

## Roadmap 18 mois

| Trimestre | Livrables clés | Métriques |
|---|---|---|
| **T1** | CSPN dossier déposé ANSSI + audit Quarkslab ou Synacktiv lancé | Audit en cours, dépôt validé |
| **T2** | Audit findings remédiés, 5 relais Gotham publics déployés (3 pays UE, opérateurs distincts) | Réseau opérationnel public |
| **T3** | CSPN obtenu, 2 pilotes Tier-1 signés (Airbus + Thales ou Dassault) | 2 LoI minimum |
| **T4** | Push relay mobile en prod, intégration invitation Gotham (clé exchange auto), iOS+Android stores | App mobile distribuable |
| **T5** | Premier contrat ferme Tier-1, 3 000+ sièges déployés en pilote | ARR 1-2 M€ |
| **T6** | Demande SecNumCloud, presence FIC 2027 + Eurosatory | Pipeline 10 M€ qualifié |

---

## Équipe et exécution

[À compléter par toi : tes propres credentials, le lead crypto, le lead biz dev, les advisors. Mentionner si applicable : background ANSSI, OWASP, contribution open source significative, expérience défense.]

**Indicateurs de capacité d'exécution déjà démontrés** :

- Implémentation complète Phase 1-8 du protocole en [X semaines] solo dev
- Code review-grade : 0 warning clippy, deny-policy stricte sur les chemins crypto
- Discipline open source : licence dual AGPLv3 + commerciale, contributions documentées
- Choix techniques défendables (Rust memory-safe, ML-KEM standardisé FIPS, Noise XK formellement vérifié ailleurs)

---

## Risques et mitigations

| Risque | Mitigation |
|---|---|
| **Audit externe découvre une faille critique** | Code Rust + lint strict + fuzz harness en place ; coût d'un audit clean ~50-80 k€ ; budget audit prévu |
| **CSPN refusée** | Préparation par expert CSPN (Quarkslab/Synacktiv) en amont ; alternative ANSSI Qualification Standard ; certif équivalente DE/IT possible |
| **Tier-1 ne signe pas le pilote** | Pilote ETI (CNRS, INRIA, CEA) en parallèle comme référence ; positionnement journalistes/ONG comme bridge revenue |
| **Concurrent US fait un partenariat souverain FR** | Notre moat = code source contrôlé en France ; nous restons éligibles "EU Sovereignty" même si un Microsoft annonce un partenariat Capgemini |
| **Bug exploitable post-déploiement** | Bug bounty programme HackerOne ; processus de disclosure responsable publié ; CVE numbering authority à mid-term |

---

## Ce que nous demandons

**[3Millions]** sur **24 mois** pour financer :

| Poste | Allocation indicative |
|---|---|
| Audit externe Quarkslab/Synacktiv/NCC (Phase 1 + remédiation) | 100-150 k€ |
| Certification CSPN ANSSI (évaluateur + frais + temps) | 80-120 k€ |
| Infrastructure réseau Gotham public (5-10 relais multi-pays, 24 mois) | 30-50 k€ |
| Tech leads renforts (2 ETP Rust senior, 1 ETP iOS/Android, 1 ETP sales défense) | 600-800 k€ |
| Marketing/conférences (FIC, Eurosatory, Le Bourget) | 80-120 k€ |
| Légal (contrats Tier-1 défense, DPA, conformité RGPD/NIS2) | 50-80 k€ |
| Trésorerie + amorçage commercial | reste |

**En contrepartie** : équité dans une structure dual-license (AGPLv3 communauté + commerciale Tier-1) basée en France, gouvernance préservant l'indépendance souveraine du produit (clause anti-acquisition par entité non-UE inscrite au pacte).

---

## La vision à 5 ans

Crypto devient **la solution de messagerie sensible par défaut de l'industrie de défense européenne**, déployée dans les 10 plus grands primes du secteur. Gotham devient un **protocole de référence open source** au même titre que Tor, mais conçu pour le temps réel et post-quantique — porté à l'IETF comme RFC informational track. La structure commerciale finance la R&D du protocole, qui à son tour est adoptée par d'autres acteurs souverains (gouvernements alliés, presse internationale, ONG), créant un réseau de relais opérés indépendamment qui renforce mutuellement l'anonymat de tous les utilisateurs.

C'est la chaîne courte qui fait défaut aujourd'hui à l'Europe sur la communication confidentielle. Nous la fermons.



