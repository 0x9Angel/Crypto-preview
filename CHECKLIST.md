# Crypto + Gotham — Master Checklist to GA

> Vue exhaustive de tout ce qui reste pour amener l'application à
> 100 % fonctionnelle et opérationnelle en production entreprise.
> `GOTHAM-CHECKLIST.md` couvre le protocole en détail ; ce fichier
> couvre **tout le reste** (produit, serveur, tests, infra, identité,
> compliance, doc, légal, marketing, distribution, ops).

**Last updated:** 2026-05-25
**Project lead:** Angel
**Target GA enterprise:** Q2 2028

**Status legend.** `[x]` = shipped on `main`. `[~]` = WIP (feature
branch, scaffold, or partial). `[ ]` = not started. `[!]` = blocker
identified, no path yet.

**Project size baseline (2026-05-25):**
- 24 983 lignes Rust · 6 106 lignes TS/TSX
- 11 crates workspace · 196 tests passent
- 15 migrations schema · Tauri 2 + React 19
- 4 workflows CI · 0 audit externe · 0 relais Gotham en production

---

## TRACK 1 — PRODUIT (Crypto messenger)

### 1.1 Desktop UI

- [x] Workspace UI v2 (Tauri 2 + React 19 + Tailwind v4)
- [x] Settings → Profile → Identity panel
- [x] Settings → Transport selector (Tor / Lokinet / Gotham)
- [x] GothamPanel (init / status / send / set_contact_pk)
- [x] Dev 3-relay self-loop bootstrap helper
- [x] Updater badge (passive probe + dot on rail button)
- [x] **1.1.1** Onboarding wizard — 5-step product tour shown on first unlock (Welcome / Privacy / Transport / First contact / Ready). Modal with Skip + Back/Next + Finish. `set_seen_onboarding` / `get_seen_onboarding` Tauri commands persist the dismissed flag so the wizard never re-appears unless the database is wiped.
- [x] **1.1.2** Recovery flow UX — **two-path recovery**: (a) **device-loss** via encrypted SQLCipher export/import (`export_db` / `import_db` + Settings UI + backup-age nag); (b) **forgotten master password** via 32-hex recovery code (`crypto-store/src/recovery.rs` — Argon2id-derived wrapping key seals the SQLCipher operating key in `recovery.json` sidecar). On profile creation a `RecoverySetupModal` shows the code with copy + "I have saved it" gate. On the login screen a "Forgot password? Use recovery code" link opens `RecoverFlowModal` calling `recover_account(code, new_password)` (PRAGMA rekey + re-seal envelope under the same code). 6 unit tests on the store side (parse format, seal/open round-trip, wrong-code AEAD reject, tampered envelope reject, entropy sanity).
- [~] **1.1.3** Multi-device sync UX — **scaffold shipped + sprint planned**: schema migration v17 adds `device_registry` table (device_id UUIDv4, name, platform, added_at, is_local, last_seen_at), Tauri commands `list_devices` / `rename_device`, Settings → Devices panel listing the local device with rename action + clear "pairing on the roadmap" disclosure. 3 new unit tests on the store side. **Pairing flow + ratchet-state sync are deliberately NOT in this scaffold** — they need a dedicated protocol design + threat-model review (multi-device = new attack surface). Full implementation sprint planned in [B10-MULTI-DEVICE-SPRINT.md](B10-MULTI-DEVICE-SPRINT.md) — ~4 weeks calendar (1 design + 2 implementation + 1 review) gated on the broader Gotham audit, earliest start Q3-Q4 2027.
- [x] **1.1.4** File attachments UI — `send_file` async Tauri command + `FileDisplay` React component + attach button in chat composer + base64 round-trip + progress events. Was wired in earlier work, now formally cocked.
- [x] **1.1.5** Voice messages — MediaRecorder (`audio/webm;codecs=opus`) → blob → base64 → `send_file` (rides on the B4 file pipeline). Mic button in chat composer: click to record, click to stop & send, right-click to cancel. Receiver renders inline `<audio controls>` via the extended `FileDisplay` component (audio MIME branch).
- [—] **1.1.6** Video / voice calls — **deferred by design — out of v1 scope** per the original product pitch. The v1 spec explicitly excludes synchronous calls; this is a product decision, not a delivery gap. Requires WebRTC + STUN/TURN + SRTP key agreement + jitter buffer + echo cancellation + UI, each its own engineering body of work. Voice messages (1.1.5) cover the asynchronous-audio use case for v1. Scheduled for a v2 release after enterprise GA.
- [x] **1.1.7** Read receipts UI (with privacy toggle) — receive side already wired (frontend tick marks via `read-receipt` event). Added global `share_read_receipts` toggle in Settings, gated in `send_read_receipt`.
- [x] **1.1.8** Typing indicators (with privacy toggle) — backend was already wired (`ChatPayload::TypingIndicator`, 6-second auto-clear). Added `share_typing` global privacy toggle in Settings, gated in `send_typing` command.
- [x] **1.1.9** Group chats (3+ participants) — 8 backend Tauri commands (`create_group`, `get_groups`, `create_group_channel`, `get_group_channels`, `get_group_members`, `get_group_messages`, `send_group_message`, `invite_to_group` + `delete_group`, `leave_group`), full UI for `viewMode === "group"` with channel sidebar, 13 frontend invoke sites. Was wired in earlier work, now formally cocked. Sender Keys ratchet (proper E2E for groups) is the open hardening item.
- [x] **1.1.10** Search across history — FTS5 trigram index (schema migration v16) + `search_messages_global` CRUD + `search_global` Tauri command + Cmd/Ctrl+K global search modal in the chat UI. 4 new tests (cross-conversation, deleted-exclusion, empty-query, FTS5-injection-safe).
- [x] **1.1.11** Export conversation — Ed25519-signed JSONL (`crypto-tauri/src-tauri/src/export.rs`, `export_conversation` Tauri command, Export icon in chat header, 6 tests passing)
- [x] **1.1.12** Notification settings — quiet hours window (`set_quiet_hours` / `get_quiet_hours` Tauri commands + UI in Settings, cross-midnight handling, 5 unit tests) + per-contact mute (`set_contact_muted` / `get_contact_muted` + Mute checkbox in contact profile panel + check in OS notification emission block).

### 1.2 Accessibility & i18n

- [ ] **1.2.1** WCAG 2.2 AA audit (contrast, keyboard nav, screen-reader)
- [ ] **1.2.2** Réduction motion (`prefers-reduced-motion`) systématique
- [ ] **1.2.3** Focus rings visibles partout (audit)
- [x] i18n FR + EN dans le site `prototype/`
- [ ] **1.2.4** i18n FR + EN dans l'app (actuellement EN-only)
- [ ] **1.2.5** i18n DE, ES, IT, PT (UE étendue)
- [ ] **1.2.6** RTL support (AR, HE) — deferred si pas de demande

### 1.3 Mobile (iOS + Android)

- [~] Targets `cfg(target_os = "android", target_os = "ios")` dans Cargo.toml
- [ ] **1.3.1** `cargo tauri android init` — initialiser le projet Android
- [ ] **1.3.2** `cargo tauri ios init` — initialiser le projet iOS
- [ ] **1.3.3** Adapter le frontend React pour viewports mobile
- [ ] **1.3.4** SQLCipher mobile : vérifier bindings sur arm64-android / aarch64-apple-ios
- [ ] **1.3.5** Keychain iOS pour stockage clé master
- [ ] **1.3.6** Android Keystore + StrongBox/TEE pour stockage clé
- [ ] **1.3.7** A5.4 battery-aware cover-traffic degradation
- [ ] **1.3.8** A5.5 backgrounded-app pause + resume
- [ ] **1.3.9** Push notifications iOS (APNS via `crypto-gotham-push`)
- [ ] **1.3.10** Push notifications Android (FCM via `crypto-gotham-push`)
- [ ] **1.3.11** App Store submission (Apple Developer Program — 99 $/an)
- [ ] **1.3.12** Play Store submission (Google Play Console — 25 $ one-time)
- [ ] **1.3.13** F-Droid submission (recommandé pour cypherpunks)

---

## TRACK 2 — PROTOCOLE (Gotham)

→ Voir [GOTHAM-CHECKLIST.md](GOTHAM-CHECKLIST.md) pour le détail.
Statut agrégé 2026-05-25 :

| Phase | Statut |
|-------|--------|
| 1 Sphinx packet | shipped (main) + A1.8 folded-KEM sur `gotham-v0.2` |
| 2 Relay binary | shipped |
| 3 Directory | shipped (main) + A3.7 anonymous refresh sur `gotham-v0.2` |
| 4 App integration | shipped (7/7 sous-items) |
| 5 Cover traffic | desktop · mobile bits sur 1.3 |
| 6 Pluggable transports | A6.1 main · A6.2–A6.5 sur `gotham-v0.2` |
| 7 Hardening | partiel main · fuzz + Kani + dudect sur `gotham-v0.2` |
| 8 Push relay | scaffolds sur `gotham-v0.2` |

**Prochain gate Gotham :** audit externe avant merge `gotham-v0.2 → main`.

---

## TRACK 3 — SERVER (crypto-server fédération)

- [x] `crypto-server` crate (2 560 LOC)
- [x] `Dockerfile` + `nullchat-server.service` (systemd)
- [x] `install-relay.sh` script
- [ ] **3.1** Déploiement effectif sur 1 instance test (jamais déployé à date)
- [ ] **3.2** Configuration HA / load balancer
- [ ] **3.3** Clustering inter-server (A6 enterprise tier)
- [ ] **3.4** Multi-tenant (workspace isolation)
- [ ] **3.5** Quotas par tenant (CPU / RAM / storage)
- [ ] **3.6** Monitoring Prometheus (counters-only, no PII)
- [ ] **3.7** Backup / restore procédures
- [ ] **3.8** Key rotation procedures documentées
- [ ] **3.9** Graceful shutdown verified under load
- [ ] **3.10** Documentation déploiement entreprise

---

## TRACK 4 — TESTING & AUDIT

### 4.1 Tests unitaires & intégration

- [x] 196 tests total project (61 gotham + 27 relay + 3 cross-crate + 105 reste)
- [x] `tests/cross_crate_e2e.rs` cross-crate gotham ↔ relay (3 tests)
- [x] `crypto-tests/tests/integration.rs` x3dh / ratchet (589 LOC)
- [x] **4.1.1** Tests d'intégration `crypto-tauri` ↔ `crypto-store` (DB round-trips) — `crypto-tests/tests/store_integration.rs` (7 tests : profile keys round-trip, full conversation lifecycle, FTS5 cross-conversation search, settings set/get round-trip, device registry upsert+list, B2 recovery chain, full migration ladder).
- [x] **4.1.2** Tests d'intégration `crypto-tauri` ↔ `crypto-server` (federation) — `crypto-tests/tests/server_integration.rs` (6 tests : Noise XX handshake, create_queue distinct IDs, send→get round-trip, GET on empty queue returns None, MAX_MSG_SIZE enforced, multi-sender same queue). Le serveur tourne in-process sur un port éphémère 127.0.0.1.
- [ ] **4.1.3** Tests d'intégration `crypto-tauri` ↔ `crypto-gotham` (full pipeline)
- [x] **4.1.4** Tests de migration schema v1 → v17 — `crypto-store/src/db.rs` mod migration_tests (6 tests : v1 baseline schema, v2 preserves v1 data, v3 settings + pending_invites, full ladder v1→v17 with fixture preservation, v16 FTS5 trigger sync on insert/update/delete, v17 device_registry idempotency).
- [ ] **4.1.5** Tests UI Playwright/WebDriver (golden path scénarios)
- [x] **4.1.6** Coverage report — `cargo llvm-cov` wired via `scripts/coverage.sh` (summary / html / lcov modes). Baseline 2026-05-25 : **63.56 %** line coverage workspace (hors tauri-app et licensor). Highlights : sealed.rs 99 % · header.rs 98 % · hybrid.rs 98 % · directory.rs 94 % · recovery.rs 92 %. Gap principal : crypto-server (jamais déployé). Doc complète dans [docs/COVERAGE.md](docs/COVERAGE.md). Cible 80 % atteignable via 4.1.1 (+10 %) et 4.1.2 (+5 %).
- [x] **4.1.7** Snapshot tests sur les composants React critiques — Vitest + @testing-library/react + jsdom configurés via [vite.config.ts](crypto-tauri/vite.config.ts). 22 tests dans [App.test.tsx](crypto-tauri/src/App.test.tsx) : renderMarkdown (parser inline), PresenceDot (mapping couleur), EmojiPicker (8 emojis fixés), FileDisplay (3 branches image/audio/download), getRtcConfig (helper WebRTC). Snapshots dans `src/__snapshots__/`.

### 4.2 Fuzz coverage

- [~] 5 fuzz targets sur `gotham-v0.2` (`header_decode`, `header_v2_decode`, `sealed_unseal`, `directory_refresh_decode`, `payload_v2_unframe`)
- [ ] **4.2.1** Merger fuzz/ dans `main` (gated sur audit)
- [ ] **4.2.2** Ajouter cible fuzz `cover_scheduler_decode`
- [ ] **4.2.3** Ajouter cible fuzz `directory_signed_decode`
- [ ] **4.2.4** Ajouter cible fuzz `relay_process_packet` (replay-cache + bad MAC)
- [ ] **4.2.5** Ajouter cible fuzz `crypto-store` migration paths
- [ ] **4.2.6** Run fuzz 24 h continu sur chaque cible (CI mensuelle)
- [ ] **4.2.7** Intégrer OSS-Fuzz (gratuit, audit continu Google) si projet OSS

### 4.3 Formal verification

- [~] 6 preuves Kani sur `gotham-v0.2` (`proofs/lib.rs`)
- [ ] **4.3.1** Merger proofs/ dans `main`
- [ ] **4.3.2** Étendre Kani aux paths `sealed_unseal` (current : header only)
- [ ] **4.3.3** Évaluer Creusot (alternative plus expressive) pour les invariants ratchet
- [ ] **4.3.4** Documenter les hypothèses de modélisation (chaque preuve = quel invariant exactement)

### 4.4 Property-based testing

- [x] `proptest` en dev-dep sur crypto-gotham-relay
- [ ] **4.4.1** Generators proptest pour `RoutingRecord`, `Header`, packets bien-formés
- [ ] **4.4.2** Propriété : `wrap(unwrap(p)) = p` pour tout packet 3 ≤ hops ≤ 5
- [ ] **4.4.3** Propriété : `seal(unseal(env)) = (sender_pk, body)` pour tout body ≤ MAX
- [ ] **4.4.4** Propriété : replay cache ne grossit jamais sans bound

### 4.5 Constant-time / side-channel

- [~] `dudect`-style Criterion benches scaffolded sur `gotham-v0.2` (`benches/timing_leak.rs`)
- [ ] **4.5.1** Merger benches dans `main`
- [ ] **4.5.2** Lancer dudect sur `unwrap_header` avec 10⁶ samples
- [ ] **4.5.3** Lancer dudect sur `unseal` avec 10⁶ samples
- [ ] **4.5.4** CI gate : pas de régression timing > 5 % entre commits
- [ ] **4.5.5** Cache-timing analysis (Microwalk / SCALib) — long terme

### 4.6 Audit externe

- [ ] **4.6.1** Quote 3 firmes : Trail of Bits · Quarkslab · Synacktiv (cible : 80-120 k€ pour audit cryptographique Gotham)
- [ ] **4.6.2** Définir le scope d'audit (Gotham seul, Crypto seul, ou les deux)
- [ ] **4.6.3** Préparer le dossier technique (architecture, threat model, build reproducible)
- [ ] **4.6.4** Signature de l'engagement
- [ ] **4.6.5** Phase d'audit (~8-12 semaines)
- [ ] **4.6.6** Remediation des findings (cible : 0 Critical, 0 High avant publication)
- [ ] **4.6.7** Publication du rapport (ou résumé NDA si client demande)
- [ ] **4.6.8** Re-audit annuel (cadence)

### 4.7 Bug bounty + responsible disclosure

- [x] `SECURITY.md` à la racine
- [ ] **4.7.1** Programme bug bounty HackerOne ou YesWeHack (pot initial : 5-10 k€)
- [ ] **4.7.2** Politique de divulgation responsable formalisée (90 jours embargo)
- [ ] **4.7.3** Hall of Fame chercheurs (page publique)
- [ ] **4.7.4** CVE Numbering Authority (CNA) registration (long terme, ~6 mois process)

---

## TRACK 5 — INFRASTRUCTURE & DEPLOYMENT

### 5.1 Relais Gotham production

- [x] 1 relais embarqué dans l'app Tauri (dev)
- [ ] **5.1.1** Choisir 3+ providers diversifiés (Hetzner DE · OVH FR · Vultr · Linode)
- [ ] **5.1.2** 1 × entry + 3 × mix + 1 × exit relais (minimum 5 nœuds)
- [ ] **5.1.3** DNS `relay-N.gotham.<domain>` avec certs Let's Encrypt
- [ ] **5.1.4** Déploiement binaires reproducibles (matching commit SHA)
- [ ] **5.1.5** Vérification reachability mutuelle des relais
- [ ] **5.1.6** Firewalls : UDP/443 + TCP/443 + SSH admin IPs seulement
- [ ] **5.1.7** `fail2ban` SSH brute-force prevention
- [ ] **5.1.8** Logging centralisé (uptime + counters seulement, jamais per-packet)
- [ ] **5.1.9** Geographic diversity dashboard
- [ ] **5.1.10** Bandwidth budget tier-tiered

### 5.2 Directory authority

- [ ] **5.2.1** Static-file host directory JSON (Cloudflare Pages / Netlify)
- [ ] **5.2.2** Cron quotidien refresh `valid_after` / `valid_until`
- [ ] **5.2.3** Clé Ed25519 authority sur YubiKey (signature offline)
- [ ] **5.2.4** Backup clé authority (Shamir split 3-of-5)
- [ ] **5.2.5** Procédure de rotation clé authority documentée

### 5.3 Monitoring (privacy-preserving)

- [ ] **5.3.1** Liveness probe par relais (HTTP /health endpoint sans métriques per-packet)
- [ ] **5.3.2** Dashboard uptime agrégé (Grafana)
- [ ] **5.3.3** Alertes > 5 min downtime (Telegram / Signal bot vers LG)
- [ ] **5.3.4** Status page publique (status.crypto.fr ou équivalent)

### 5.4 Push relay deployment

- [~] `crypto-gotham-push` crate sur `gotham-v0.2`
- [ ] **5.4.1** Déploiement push relay sur 1 instance (UE souveraine)
- [ ] **5.4.2** APNS certificate Apple Developer (99 $/an)
- [ ] **5.4.3** FCM credentials Firebase project
- [ ] **5.4.4** Validation real-device iOS + Android (latence < 5 s cible)

---

## TRACK 6 — IDENTITY & AUTH

- [x] OIDC SSO (Google Workspace, Entra ID, Okta, Auth0, Keycloak)
- [x] SCIM 2.0 user + group provisioning
- [x] Argon2id master password (12 chars, 2 character classes)
- [x] Exponential unlock back-off (3 free → 5 s → 30 s → 2 min → 10 min)
- [ ] **6.1** SAML 2.0 (roadmap — demandé secteur défense FR)
- [ ] **6.2** WebAuthn (roadmap — biométrie navigateur)
- [ ] **6.3** FIDO2 / YubiKey hardware token second-factor
- [ ] **6.4** Smart card PKCS#11 (long-term identity storage)
- [ ] **6.5** Secure Enclave macOS pour clés éphémères
- [ ] **6.6** Recovery flow : perte device sans backup
- [ ] **6.7** Recovery flow : perte master password (avec quorum admin)
- [ ] **6.8** Offboarding flow : révocation immédiate via SCIM

---

## TRACK 7 — COMPLIANCE & CERTIFICATIONS

### 7.1 Aujourd'hui

- [x] **GDPR / RGPD** : zero-metadata architecture by design
- [x] **e-Evidence Reg.** : compatible (self-hosted + sub-processor doc)
- [x] DPA template fourni dans le repo
- [x] DPIA template fourni dans le repo

### 7.2 En cours

- [~] **ISO 27001:2022** : 42 % d'Annex A mappé · Stage 1 audit cible Q1 2027
- [ ] **7.2.1** Compléter mapping 100 % Annex A
- [ ] **7.2.2** Choisir auditeur (Bureau Veritas / AFNOR / BSI)
- [ ] **7.2.3** Stage 1 audit
- [ ] **7.2.4** Stage 2 audit
- [ ] **7.2.5** Certification émise

### 7.3 SOC 2

- [~] TSC 1, 2, 4 controls mappés
- [ ] **7.3.1** Type I evidence pipeline (Drata / Vanta)
- [ ] **7.3.2** SOC 2 Type I audit Q4 2026
- [ ] **7.3.3** SOC 2 Type II audit Q2 2027

### 7.4 CSPN ANSSI (FR — clé pour défense)

- [ ] **7.4.1** Identifier le scope (messaging + Gotham probable)
- [ ] **7.4.2** Choisir évaluateur (Quarkslab / Synacktiv / Amossys / Lexfo / Oppida)
- [ ] **7.4.3** Préparer dossier technique (architecture, threat model, crypto rationale)
- [ ] **7.4.4** Soumettre demande ANSSI
- [ ] **7.4.5** Phase évaluation (~3-6 mois)
- [ ] **7.4.6** Adresser findings + re-eval si besoin
- [ ] **7.4.7** Recevoir CSPN — enregistrer produit publiquement

**Budget CSPN :** 50-80 k€ + ~12 mois calendrier.

### 7.5 Long terme

- [ ] **7.5.1** Common Criteria EAL2+ OU ANSSI Qualification Standard (décider)
- [ ] **7.5.2** FIPS 140-3 profile (roadmap 2027)
- [ ] **7.5.3** NIS2 / DORA gap analysis
- [ ] **7.5.4** Incident response plan (4-h notification ENISA-equivalent)
- [ ] **7.5.5** SBOM signé pour chaque release
- [ ] **7.5.6** SecNumCloud (seulement si offre SaaS managée)

---

## TRACK 8 — DOCUMENTATION

- [x] `README.md`
- [x] `GOTHAM.md` (spec protocole)
- [x] `GOTHAM-CHECKLIST.md`
- [x] `GOTHAM-DOSSIER.md` (référence ultra-détaillée 1611 lignes)
- [x] `PITCH.md`
- [x] `SECURITY.md`
- [x] `docs/MANUAL-QA.md`
- [x] `docs/REPRODUCIBLE-BUILDS.md`
- [x] API reference `cargo doc --no-deps` clean (0 warnings)
- [ ] **8.1** `CHANGELOG.md` (per-version changes)
- [ ] **8.2** `GOTHAM-OPSEC.md` (operator guide — security best practices for relay admins)
- [ ] **8.3** `GOTHAM-USERGUIDE.md` (end-user documentation)
- [ ] **8.4** `GOTHAM-DEPLOYMENT.md` (how to operate a relay)
- [ ] **8.5** `GOTHAM-THREAT-MODEL.md` (extended threat model with attack trees)
- [ ] **8.6** `CONTRIBUTING.md` avec security-disclosure section
- [ ] **8.7** Code of Conduct (Contributor Covenant)
- [ ] **8.8** Gouvernance doc (maintainers, decision process)
- [ ] **8.9** **Paper académique Gotham** (~20 pages, USENIX/IEEE S&P format) — voir track 4.6
- [ ] **8.10** Blog post "Introducing Gotham" (publication post-audit)
- [ ] **8.11** Soumission RFC IETF informational track (long terme)

---

## TRACK 9 — LEGAL & BUSINESS STRUCTURE

### 9.1 Structure juridique

- [ ] **9.1.1** Création **SASU** (avocat startup : Hashtag / Aurore / Goodwin Paris — ~1000-1500 €)
- [ ] **9.1.2** Capital social (1 € à 1000 €)
- [ ] **9.1.3** Cap table initial (100 % founder au départ)
- [ ] **9.1.4** Vesting founder : 4 ans, cliff 1 an
- [ ] **9.1.5** **Cession IP** repos perso → SASU (acte notarié)
- [ ] **9.1.6** RIB pro + compte bancaire entreprise (Qonto / Shine / banque classique)

### 9.2 Propriété intellectuelle

- [ ] **9.2.1** Dépôt marque **Crypto** à l'INPI (classes 9, 38, 42 — ~250 €)
- [ ] **9.2.2** Dépôt marque **Gotham** à l'INPI (classes 9, 38, 42 — ~250 €)
- [ ] **9.2.3** Extension EU via EUIPO (~850 € classes 9+42)
- [ ] **9.2.4** Audit brevetabilité (Gotham folded-KEM peut-être) — avocat brevet ~2-5 k€
- [ ] **9.2.5** Dépôt brevet sélectif si pertinent (~5-15 k€ par brevet)
- [ ] **9.2.6** Compliance open-source (AGPLv3 + dépendances licences — `cargo-deny`)

### 9.3 Contrats & policies

- [ ] **9.3.1** CGV / Terms & Conditions (FR + EN — avocat)
- [ ] **9.3.2** Privacy Policy (FR + EN, GDPR-compliant)
- [ ] **9.3.3** DPA template signable par les clients (rev par avocat)
- [ ] **9.3.4** DPIA template (validation CNIL pas obligatoire mais possible)
- [ ] **9.3.5** SLA template (99,9 % uptime Enterprise tier)
- [ ] **9.3.6** Sub-processor list publique (Cloudflare, Hetzner, etc.)
- [ ] **9.3.7** Contrats employés (avec clause IP cession + non-concurrence)
- [ ] **9.3.8** NDA template investisseurs / clients
- [ ] **9.3.9** Designated DPO (Data Protection Officer) — peut être fractional

### 9.4 Assurances

- [ ] **9.4.1** RC pro (responsabilité civile professionnelle)
- [ ] **9.4.2** Cyber assurance (couverture incident sécurité)
- [ ] **9.4.3** D&O insurance (Directors & Officers — quand board en place)

### 9.5 Comptabilité

- [ ] **9.5.1** Expert-comptable retenu (Dougs / Pennylane / cabinet traditionnel)
- [ ] **9.5.2** Logiciel facturation (Pennylane / Stripe Invoicing)
- [ ] **9.5.3** Numéro SIREN + TVA intracommunautaire

---

## TRACK 10 — MARKETING & SALES

### 10.1 Présence digitale fondamentale

- [ ] **10.1.1** Acheter **domaine** (`crypto.fr` si dispo · sinon `crypto-protocol.com` / `crypto-suite.eu`) — OVH/Gandi/Infomaniak
- [ ] **10.1.2** **Email pro** : ProtonMail Business avec domaine custom (~7-13 €/mois)
- [ ] **10.1.3** Créer adresses : `enterprise@`, `security@`, `press@`, `investors@`, `contact@`
- [ ] **10.1.4** **Clé PGP** générée (Ed25519 ou RSA 4096) + publiée sur `keys.openpgp.org`
- [ ] **10.1.5** **Onion v3** : générer vraie hostname `tor` (56 caractères base32)
- [ ] **10.1.6** Calendly compte pour discovery calls
- [ ] **10.1.7** LinkedIn pro (sans "founder of Crypto" tant que stealth)
- [ ] **10.1.8** Compte Twitter/X et Mastodon (instance infosec.exchange)

### 10.2 Repos publics GitHub

- [x] README public `github.com/0x9Angel/Crypto` (teaser only)
- [x] README public `github.com/0x9Angel/gotham-protocol` (teaser only)
- [x] Profil GitHub `@0x9Angel` actualisé
- [ ] **10.2.1** Activer Issues / Discussions sur les 2 repos
- [ ] **10.2.2** Labels standards (bug, feature, security, docs)
- [ ] **10.2.3** PR / Issue templates
- [ ] **10.2.4** Hall of Fame contributors

### 10.3 Site vitrine

- [~] `prototype/` site v2 (refonte Crypto + Gotham) — pas déployé
- [ ] **10.3.1** Appliquer P0/P1/P2 du brief de révision (URL Crypto fix, etc.)
- [ ] **10.3.2** og-image.png 1200×630
- [ ] **10.3.3** Déployer sur Cloudflare Pages / Netlify (uniquement quand SASU créée)
- [ ] **10.3.4** Configurer domaine custom
- [ ] **10.3.5** Analytics privacy-respecting (Plausible / Fathom) — pas Google Analytics

### 10.4 Dossier investisseur (4 artefacts)

- [ ] **10.4.1** **One-pager** PDF 1 page A4 (300 mots)
- [ ] **10.4.2** **Pitch deck** PDF 12-16 slides
- [ ] **10.4.3** **Memo investisseur** PDF 5-8 pages narratif
- [ ] **10.4.4** **Data room** structurée (00_Overview → 08_Diligence_QA)
- [ ] **10.4.5** Disclaimer AMF/ESMA validé par avocat
- [ ] **10.4.6** Liste 50-70 fonds qualifiés (Tier 1/2/3 + corporate VCs)
- [ ] **10.4.7** Pipeline outreach (cold email → first call → memo → DD → term sheet)

### 10.5 Demo & média

- [ ] **10.5.1** Demo video 60-90 s (vidéo teaser brief Claude Design)
- [ ] **10.5.2** Demo video 5 min (walkthrough produit pour RSSI)
- [ ] **10.5.3** Screenshots haute-def (light + dark mode)
- [ ] **10.5.4** Press kit (wordmark, palette, 3 phrases d'accroche, bio founder)
- [ ] **10.5.5** Vidéo institutionnelle (3-4 min — pour FIC / Eurosatory)

### 10.6 Présence physique

- [ ] **10.6.1** Participation **FIC** Lille (janvier) — visiteur 2027, exposant 2028 ?
- [ ] **10.6.2** Participation **Eurosatory** Paris (juin, années paires) — 2028 cible
- [ ] **10.6.3** Participation **FOSDEM** Bruxelles (février — communauté OSS)
- [ ] **10.6.4** Participation **USENIX Security** (août — académique)
- [ ] **10.6.5** Participation **les Assises de la Sécurité** Monaco (octobre — RSSI FR)

### 10.7 Sales pipeline

- [ ] **10.7.1** 1-3 **pilotes signés sous NDA** (peu importe qui, mais réels)
- [ ] **10.7.2** Première lettre d'intention (LOI)
- [ ] **10.7.3** Premier contrat payant (ARR > 0 €)
- [ ] **10.7.4** Référencement UGAP (Union des Groupements d'Achats Publics)
- [ ] **10.7.5** ANSSI "Visa de sécurité" (séparé de CSPN)
- [ ] **10.7.6** Pipeline Tier-1 : Airbus, Thalès, Dassault — formal RFP track
- [ ] **10.7.7** Pipeline Tier-2 : Naval Group, MBDA, Safran, Nexter
- [ ] **10.7.8** Pipeline gouvernement (post-déplacement Tchap — long terme)

---

## TRACK 11 — DISTRIBUTION

### 11.1 Linux

- [x] **Arch Linux** : PKGBUILD shipped (`dist-arch/`)
- [x] AppImage construit (`pkgbuild/Crypto_0.1.0_amd64.AppImage`)
- [ ] **11.1.1** **Debian / Ubuntu** `.deb` package + APT repository
- [ ] **11.1.2** **Fedora / RHEL** `.rpm` package + DNF repository
- [ ] **11.1.3** **Flatpak** (universel Linux)
- [ ] **11.1.4** **Snap** (Ubuntu Store) — optionnel
- [ ] **11.1.5** **Nix** package (NixOS / nixpkgs) — communauté technique
- [ ] **11.1.6** **AUR** publication (`yay -S crypto-messenger`)

### 11.2 macOS

- [ ] **11.2.1** Apple Developer Program ($99/an)
- [ ] **11.2.2** Build `.dmg` signé Developer ID
- [ ] **11.2.3** Notarization Apple (anti-Gatekeeper)
- [ ] **11.2.4** Hardened runtime + entitlements minimum
- [ ] **11.2.5** Auto-update via Sparkle ou Tauri updater
- [ ] **11.2.6** **Mac App Store** submission (optionnel — sandboxing restrictif)

### 11.3 Windows

- [ ] **11.3.1** Build `.msi` Wix toolset
- [ ] **11.3.2** Code-signing certificate EV (~300 €/an Sectigo / DigiCert)
- [ ] **11.3.3** SmartScreen reputation (immediate trust)
- [ ] **11.3.4** Microsoft Store submission (optionnel)
- [ ] **11.3.5** Auto-update via Tauri updater

### 11.4 Mobile (track 1.3)

→ Voir 1.3.11–1.3.13

### 11.5 Auto-update infrastructure

- [x] Updater badge UI (passive probe)
- [ ] **11.5.1** Endpoint update.crypto.<domain> (signed manifests)
- [ ] **11.5.2** Delta updates (vs full re-download)
- [ ] **11.5.3** Rollback safety (atomic install)
- [ ] **11.5.4** Cadence release : `stable` (3 mois) · `beta` (mensuelle) · `nightly` (auto)

### 11.6 Supply chain security

- [x] Reproducible builds profile (`Cargo.toml` `[profile.release]`)
- [x] `docs/REPRODUCIBLE-BUILDS.md`
- [x] Tier-4 reproducible builds (commit `94017c6`)
- [ ] **11.6.1** SBOM auto-généré à chaque release (`cargo-sbom`)
- [ ] **11.6.2** Signature Sigstore / `cosign` sur les binaries
- [ ] **11.6.3** SLSA Level 3 attestations (GitHub Actions)
- [ ] **11.6.4** Witness arbiter client documenté (déjà code, doc manquante)
- [ ] **11.6.5** Publication checksums + signatures par release

---

## TRACK 12 — OPERATIONS & SUPPORT

### 12.1 Support client

- [ ] **12.1.1** Email support (`support@<domain>`)
- [ ] **12.1.2** Ticketing system (Plain / Front / Linear)
- [ ] **12.1.3** Knowledge base (Notion public / GitBook)
- [ ] **12.1.4** Status page publique
- [ ] **12.1.5** SLA monitoring + incident communication
- [ ] **12.1.6** Customer success manager (recrutement seed)

### 12.2 Incident response

- [ ] **12.2.1** Incident response plan documenté
- [ ] **12.2.2** Runbooks par scénario (key compromise, relay down, DB corruption, etc.)
- [ ] **12.2.3** On-call rotation (PagerDuty / OpsGenie)
- [ ] **12.2.4** Post-mortem template
- [ ] **12.2.5** Backup et disaster recovery procedures testées

### 12.3 Security operations

- [ ] **12.3.1** Vulnerability disclosure intake formalisé
- [ ] **12.3.2** CVE assignment workflow (via CNA si registré ou via MITRE direct)
- [ ] **12.3.3** Patch release SLA (Critical : 7 jours · High : 30 jours)
- [ ] **12.3.4** Customer notification process pour CVE

---

## CRITICAL PATH TO GA Q2 2028

Vue ordonnée des blockers absolus (sans lesquels la GA n'a pas lieu).

| Phase | Trimestre | Blocker | Dépendance |
|-------|-----------|---------|------------|
| **P1** | **Q3-Q4 2026** | Création SASU · IP cession · email pro · marques INPI | 9.1, 9.2, 10.1 |
| **P1** | **Q3-Q4 2026** | 1-2 pilotes signés NDA | 10.7.1 |
| **P1** | **Q4 2026** | Audit kickoff scoping signed (Trail of Bits ou Quarkslab) | 4.6.1-4.6.4 |
| **P2** | **Q1 2027** | **Seed close 3 M€** | tous artefacts dossier 10.4 |
| **P2** | **Q1-Q2 2027** | Premiers hires (CTO/CSO + 3 Rust seniors) | seed close |
| **P2** | **Q1 2027** | SOC 2 Type I evidence pipeline (Drata / Vanta) | seed close |
| **P3** | **Q2-Q3 2027** | Audit externe Gotham complete | 4.6 |
| **P3** | **Q2-Q3 2027** | Merge `gotham-v0.2 → main` (post-audit) | 4.6.6 |
| **P3** | **Q3 2027** | Déploiement 5 relais production multi-régions | 5.1 |
| **P3** | **Q3-Q4 2027** | CSPN ANSSI submitted | 7.4 |
| **P4** | **Q4 2027** | Mobile builds iOS + Android | 1.3 |
| **P4** | **Q4 2027** | Push notifications validées real-device | 5.4 |
| **P4** | **Q4 2027** | Documentation utilisateur + opérateur complète | 8.2-8.5 |
| **P4** | **Q1 2028** | Premiers clients payants signés | 10.7.3 |
| **P5** | **Q2 2028** | **GA Enterprise** | tout ce qui précède |
| **P5** | **Q2 2028** | SOC 2 Type II + CSPN reçus | 7.3.3, 7.4.7 |

---

## Synthèse coût estimé sur 24 mois (seed close → GA)

| Poste | Estimation |
|-------|-----------|
| Audit cryptographique externe (Trail of Bits/Quarkslab) | 80-120 k€ |
| SOC 2 Type I + II (Drata/Vanta + auditeur) | 40-80 k€ |
| CSPN ANSSI | 50-80 k€ |
| Infrastructure relais (5 nœuds × 24 mois × ~15 €/mois) | ~1,8 k€ |
| Domaine + email pro + Calendly + outils | ~2 k€ |
| Marques INPI + EUIPO + brevets sélectifs | 3-10 k€ |
| Avocats (création SASU + contrats + PI) | 8-15 k€ |
| Salaires équipe (6 Rust + 2 SRE + 1 cryptographe × 24 mois) | ~1,8 M€ |
| Salaires CSM + Sales (2 personnes × 18 mois) | ~250 k€ |
| Bug bounty pot initial | 10-20 k€ |
| Conférences (FIC, Eurosatory, Assises) | ~30 k€ |
| Code-signing certs + Apple/Google dev | ~1 k€ |
| Insurance (RC pro + cyber + D&O) | 10-20 k€ |
| **Total estimé** | **~2,4 M€** |

**→ Cohérent avec une levée 3 M€** (marge de 600 k€ pour imprévus + extension runway).

---

## Compte global tâches à date

| Statut | Count |
|--------|-------|
| `[x]` shipped | **45** |
| `[~]` WIP (feature branch / scaffold) | **18** |
| `[ ]` not started | **~210** |
| `[!]` blockers identifiés sans path | **0** |

**Avancement projet global :** ~17 % shipped, ~7 % en cours, ~76 % à faire.

L'app est **techniquement opérationnelle aujourd'hui** (code messagerie + protocole shippé sur main) mais **non commercialement opérationnelle** (zéro structure, zéro déploiement public, zéro audit, zéro client).

Le chemin de GA est clair : ~18 mois de travail séquencé, ~2,4 M€ de budget, équipe de 8-10 personnes à constituer post-seed.
