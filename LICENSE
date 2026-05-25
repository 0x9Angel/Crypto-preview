# LICENSE — Crypto & Gotham

This project is **dual-licensed**. You may choose **one** of the following
license tracks. By using, modifying, or distributing this software you
accept the terms of the track you select.

---

## Track 1 — Open-source (default)

**SPDX-License-Identifier:** `AGPL-3.0-or-later`

The full text of the GNU Affero General Public License version 3 is
available at <https://www.gnu.org/licenses/agpl-3.0.txt>. A verbatim
copy will be added to this repository as `LICENSE-AGPL.txt` before the
first public release. Until that file lands, the canonical text at the
GNU URL above governs.

**Key consequence of AGPLv3** : any **network use** of a modified version
of this software (e.g. running a fork as a hosted SMP / Gotham relay
service) requires you to publish the corresponding source code under the
same license. This is the trigger that distinguishes AGPL from GPL —
"running it" counts as "distributing it" for hosted deployments.

If you are an individual, an OSS project, a research lab, a journalist,
an NGO, a public-sector entity, or an organisation comfortable with the
AGPLv3 source-disclosure trigger, **the AGPLv3 track is the right
one for you**.

---

## Track 2 — Commercial

**SPDX-License-Identifier:** `LicenseRef-Crypto-Commercial`

If your intended use is incompatible with the AGPLv3 source-disclosure
trigger — typically because you are shipping a closed-source product
that embeds or links against this software, or you are operating a
hosted service whose customer agreements forbid open-source disclosure
— you need the commercial license track instead.

The terms of the commercial license are described honestly in
`LICENSE-COMMERCIAL.md`. **The commercial track is not yet operational:**
no pricing has been published, no purchase channel exists, no commercial
licensee has been signed. We will not lie by listing a paid licence we
cannot today deliver. The document exists so commercial users can plan
ahead.

To express interest in the commercial track:
`Crypto.app.organisation@protonmail.com`.

---

## Per-file SPDX headers

Every source file in this repository carries an SPDX-License-Identifier
comment at the top:

```rust
// SPDX-License-Identifier: AGPL-3.0-or-later OR LicenseRef-Crypto-Commercial
```

The `OR` between the two identifiers is the standard SPDX expression for
a dual license. It means "you may use this file under EITHER of these
two licenses, your choice".

The Gotham crates use a parallel pair:

```rust
// SPDX-License-Identifier: AGPL-3.0-or-later OR LicenseRef-Gotham-Commercial
```

This split lets us, in principle, license the messenger and the mixnet
to different commercial counterparties — but in practice both refer to
the same `LICENSE-COMMERCIAL.md` terms today.

---

## NO WARRANTY

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, AND NON-INFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES, OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT, OR OTHERWISE, ARISING FROM, OUT OF, OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

**Specifically — and we say this without irony**: this software has
never been deployed in production. It has not been audited by an
independent third party. Its cryptographic protocols are designed
against well-defined threat models documented in `THREAT-MODEL.md`, but
no protocol design is provably immune to flaws discovered later. **Do
not bet a life on this software in its current state.**

---

*This file replaces a stub `LICENSE` that previously contained the MIT
license text. That stub was an artefact of an early scaffolding template
and never reflected the project's intended licensing. The dual AGPLv3 +
commercial pattern has been in the per-file SPDX headers and the project
documentation since 2026-Q1; this `LICENSE` file is now aligned with
that intent.*
