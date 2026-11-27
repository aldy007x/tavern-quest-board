# Gnome Tavern — Gno.land research & deployment plan

Researched 2026-11-27 from official sources. This document backs phase 1
(local code + tests) and gives the exact phase-2 path to a live deployment
on the Onyx testnet for the DoraHacks "Gnome Tavern Hackathon | Gno.land"
(submission deadline 18 Dec 2026, 08:00 WIB).

## 1. What Gno.land is

Gno.land is a Layer-1 blockchain whose smart contracts are written in
**Gno**, a deterministic variant of Go: Go syntax minus what cannot run
on-chain (no goroutines, no `os`, no networking; only standard-library or
`gno.land/...` imports).

- **Realms** (`gno.land/r/<namespace>/<name>`) are stateful contracts:
  top-level variables persist to chain state after each transaction.
- **Packages** (`gno.land/p/...`) are stateless libraries. Pure packages
  cannot import realms.
- Each project directory carries a `gnomod.toml`:
  `module = "gno.land/r/myname/myrealm"` and `gno = "0.9"`.
- Stateful entry points are **crossing functions**: first parameter
  `cur realm`. Callers invoke them with `cross(cur)` in code, or via
  `gnokey maketx call` (the crossing happens automatically).
  Inside a crossing function, `cur.Previous()` identifies the immediate
  caller (address + whether it is a direct user call), and
  `cur.Address()` is the realm's own address.
- `Render(path string) string` returns Markdown that **gnoweb** renders
  as the realm's web page; the same page's **Actions** tab generates
  ready-to-run `gnokey` commands for every exported crossing function.

Sources:
- Getting started: https://docs.gno.land/builders/getting-started/
- Interacting with gnokey: https://docs.gno.land/users/interact-with-gnokey/

## 2. Toolchain

Three binaries (plus `gnoweb`): `gno` (fmt/lint/test/mod), `gnokey`
(keys, queries, transactions), `gnodev` (local devnet with hot reload,
pre-funded accounts, web UI on http://localhost:8888).

Official install (one-line installer, downloads prebuilt binaries into
`$HOME/.gno/bin`; supports `--version <tag>`, `--full`, `--from-source`):

```sh
curl -fsSL https://raw.githubusercontent.com/gnolang/gno/master/misc/install.sh | sh
```

Installed here (2026-11-27): **gno v1.8.0, gnokey v1.8.0, gnodev, gnoweb**
at `~/workspace/gnome-tavern/tools/bin/` (persistent workspace path,
`GNO_INSTALL_DIR` override). All four binaries execute correctly
(`gno version`, `gnokey version`, `gnodev --help` verified).

### Toolchain status — RESOLVED 2026-11-27 (see TEST-RESULTS.md)

Aldiyawan approved fetching the official source. The official `gnolang/gno`
**v1.8.0** tag archive was downloaded and extracted to
`tools/gno-src-v1.8.0/`; with `GNOROOT` pointed there, the full suite runs:
**14/14 unit tests + 2/2 filetests PASS, lint and fmt clean.** The flagged
sparse clone in `tools/gno-src/` was not used. Details: `TEST-RESULTS.md`.
The original blocker description follows for the record.

### Toolchain status — the original phase-1 blocker (historical)

The prebuilt binaries are not self-contained: `gno fmt/lint/test`
resolve the standard library from `$GNOROOT/gnovm/stdlibs` on disk
(`gno env` reports the build machine's baked-in root, `/__w/gno/gno`;
the `GNOROOT` environment variable overrides it — verified: pointing it
at an empty directory changes the resolution paths accordingly). The
installer ships binaries only, no stdlib tree.

A shallow sparse clone of `gnolang/gno` was made to
`~/workspace/gnome-tavern/tools/gno-src/` for documentation purposes,
and its example/stdlib sources were read as text (they informed the
code patterns in §6). However, that clone's tool result was flagged by
an automated execution-risk check, with instructions not to run, build,
test, import or load its content. Pointing `GNOROOT` at it would load
its stdlib into the toolchain, so **it is deliberately NOT used as
GNOROOT**, and `gno test` has therefore not been executed yet. The realm
code and tests are written and manually reviewed line-by-line against
the stdlib sources, but they are **not yet test-verified**.

Unblock options (any one suffices):

1. **Approve using a Gno source tree as `GNOROOT`** (the existing clone,
   or a fresh official release source download). Then:
   `export GNOROOT=<tree>` and `gno test ./...` in `questboard/`.
   This is the fastest route (minutes).
2. Re-run the installer with `--from-source` (needs a Go toolchain,
   which the sandbox also lacks; heavier).
3. Use the hosted **Playground** (play.gno.land, linked from the docs)
   to compile/test the realm in a browser, and deploy via `gnokey`
   (which does not need the stdlib tree). Slower, manual, but zero
   local installs.

## 3. Networks (verified against the official networks page)

| Network | Chain ID | RPC endpoint | Notes |
|---|---|---|---|
| Local devnet (gnodev) | `dev` | http://localhost:26657 | keys auto-funded, no faucet needed |
| **Onyx / Test17** | `onyx-1` | https://rpc.onyx.testnets.gno.land:443 | **the hackathon testnet** |
| Mainnet | `gnoland-1` | https://rpc.gno.land:443 | launched 2026-09-12; **no faucet — do not use** |

- `https://rpc.latest.testnets.gno.land:443` always points at the
  current testnet (today Onyx); web UI at `https://latest.testnets.gno.land`.
- Onyx launched 2026-09-28, runs mainnet's code and package set (every
  mainnet release is rehearsed there first), has a faucet and open
  transfers. A realm deployed on Onyx carries over across its in-place
  upgrades.
- Package-acceptance diagnostics: `https://gpas.latest.testnets.gno.land`
  (look up the exact package path if a deploy does not appear).

Source: https://docs.gno.land/resources/gnoland-networks/

## 4. Money mechanics

- Denomination: `ugnot`; 1 GNOT = 1,000,000 ugnot.
- Gas: floor price 1 ugnot per 1000 gas, so `-gas-fee` must be at least
  `-gas-wanted / 1000` ugnot (e.g. 2,000,000 gas ⇒ ≥ 2000 ugnot fee).
- **Storage deposit:** every byte of code/state a transaction adds
  locks **100 ugnot**, refunded when the storage is freed. A small
  realm deploy (~3.2 KB in the docs' example) locked ~318,700 ugnot
  (~0.32 GNOT) on top of a ~5,000 ugnot fee. Budget accordingly:
  one faucet claim should cover a realm deploy; keep
  `-max-deposit <amount>ugnot` in mind as a safety cap.
- Faucet: **faucet.gno.land** (linked from the Getting Started guide) —
  paste the `g1…` address, select the current testnet (Onyx), submit;
  tokens arrive in seconds; rate-limited per address.

## 5. Namespaces

- **Address-based (zero setup):** every account can deploy under
  `gno.land/r/<its-own-g1-address>/…` and nobody else can. This is the
  recommended phase-2 path.
- Named namespaces (`gno.land/r/<name>/…`) require registering the name
  on-chain first — unnecessary for the hackathon.
- Our local module path is `gno.land/r/gnometavern/questboard`; at
  deploy time only the `gnomod.toml` module line (and the `RealmPath`
  constant in `render.gno`) change to the address-based path.

## 6. Payment & security patterns applied

From the official example `examples/gno.land/r/demo/disperse` in
https://github.com/gnolang/gno and the repo's own agent guidance
(`AGENTS.md`, `docs/resources/gno-ai-contract-review.md`;
web version: https://docs.gno.land/resources/gno-ai-contract-review/):

- Verify payment with the send envelope, never a parameter:
  require `cur.Previous().IsUserCall()` (NOT `IsUser()`, which also
  accepts `maketx run` ephemeral realms that can pre-consume the
  envelope), then read the attached coins via
  `chain/runtime/unsafe.OriginSend()`. Coins sent with a call land at
  the realm's address; escrow = holding them + internal accounting.
- Pay out with `banker.NewBanker(banker.BankerTypeRealmSend, cur)`
  and `SendCoins(realmAddr, to, chain.NewCoins(chain.NewCoin("ugnot", amt)))`.
  Never persist a banker in state (it is an irrevocable spending
  capability); construct it per call.
- Checks-effects-interactions: settle status/accounting before
  `SendCoins`.
- Read single-denom balances with `GetCoin(addr, "ugnot")`, never
  `GetCoins` on a caller-supplied address (cost grows with junk denoms
  anyone can send that address).
- Caller identity always from `cur.Previous()`, never from an
  `address` function parameter.
- Sanitize user text in `Render` (ours: a small local sanitizer —
  strips markdown control characters — because the project is
  stdlib-only; the ecosystem alternative is
  `gno.land/p/nt/markdown/sanitize/v0`).
- Emit `chain.Emit(...)` events for every state transition
  (QuestPosted / QuestAccepted / QuestCompleted / QuestCancelled /
  QuestRefunded) for indexers and the demo.

Test-side pattern (from the disperse filetests): filetests use
`testing.IssueCoins`, `testing.SetOriginSend` and
`testing.SetRealm(testing.NewUserRealm(addr))` to simulate funded
users, send envelopes and caller frames end-to-end.

## 7. Design of the Tavern Quest Board (phase 1 output)

One realm, `questboard` (see README.md for the full description):

- Lifecycle: Open → Accepted → Completed (escrow paid to adventurer);
  Open → Cancelled (refund); Open|Accepted → Expired via ClaimRefund
  after the deadline (refund).
- Rewards in ugnot, exact-payment rule (attached send must equal the
  reward), min 1000 ugnot, deadline 1 s–90 days out.
- Storage: in-realm slice of quests (ID = index+1), plus an
  `escrowed` counter; invariant `escrowed == Σ rewards of
  Open/Accepted quests` asserted in tests.
- `Render` routes: `""` (board + stats + how-to with copy-paste
  gnokey commands), `open`, `all`, `quest/<id>`.
- Tests: 14 unit tests driving the internal logic with a fake
  `chain/banker.Banker` (happy paths, every guard, payment mismatch,
  escrow invariant, rendering + sanitization, GNOT formatting) and
  2 filetests for the crossing layer (post→accept→complete;
  post→cancel). **Executed and green 2026-11-27 — see TEST-RESULTS.md.**

## 8. Phase-2 deployment plan (Onyx testnet, exact steps)

Prerequisites: GNOROOT unblocked (§2) and `gno test ./...` green;
`gnokey` from `tools/bin` on PATH. All commands testnet-only.

```sh
export PATH="$HOME/workspace/gnome-tavern/tools/bin:$PATH"
export REMOTE=https://rpc.onyx.testnets.gno.land:443

# 1. Create the deployer key (development-only; save the mnemonic offline)
gnokey add tavern
ADDR=$(gnokey list | grep tavern)   # note the g1... address

# 2. Fund it: open faucet.gno.land, paste $ADDR, network = Onyx, submit.
gnokey query bank/balances/<ADDR> -remote $REMOTE   # expect a ugnot balance

# 3. Point the module at the address-based namespace
#    questboard/gnomod.toml:  module = "gno.land/r/<ADDR>/questboard"
#    questboard/render.gno:   RealmPath  = "gno.land/r/<ADDR>/questboard"
#    (gno.land link prefix in render.gno tables: /r/<ADDR>/questboard)

# 4. Deploy
cd questboard
gnokey maketx addpkg \
  -pkgpath "gno.land/r/<ADDR>/questboard" \
  -pkgdir . \
  -gas-fee 10000ugnot -gas-wanted 10000000 \
  -max-deposit 2000000ugnot \
  -chainid onyx-1 -remote $REMOTE \
  tavern
# Expect "OK!" + PKGPATH. If the realm does not appear within a minute,
# check https://gpas.latest.testnets.gno.land for the refusal reason.

# 5. Verify in gnoweb
#    https://latest.testnets.gno.land/r/<ADDR>/questboard

# 6. Seed + exercise it (deadline = now + 1 day, unix seconds)
gnokey maketx call -pkgpath "gno.land/r/<ADDR>/questboard" -func PostQuest \
  -args "Slay the spam goblins" -args "Clear the tavern cellar of spam goblins." \
  -args 5000000 -args <deadline-unix> -send 5000000ugnot \
  -gas-fee 2000ugnot -gas-wanted 2000000 -chainid onyx-1 -remote $REMOTE tavern
# AcceptQuest / CompleteQuest / CancelQuest / ClaimRefund analogously
# (AcceptQuest must come from a second key: gnokey add adventurer).
```

Wallet/address notes for the DoraHacks submission form:

- "Gno.land testnet deployment link (gnoweb URL)":
  `https://latest.testnets.gno.land/r/<ADDR>/questboard`
- "Realm or package path": `gno.land/r/<ADDR>/questboard`
- "Gno.land wallet address for prize payout": the same `<ADDR>`
  (a Gno address is required for GNOT prizes; create it with
  `gnokey add` as above and keep the mnemonic safe — it is the
  payout wallet).
- A second key (`adventurer`) is only needed to demo the full
  lifecycle on testnet.

## 9. Hackathon checklist mapping

| Requirement | Status |
|---|---|
| ≥1 realm/package deployed on Onyx + verifiable | phase 2 (§8) |
| Public GitHub repo + README | repo layout ready (`README.md`, MIT `LICENSE`); push is phase 2 |
| Demo video | phase 3 (must include audio narration per Aldiyawan's standing preference) |
| Submission in English | ✓ all docs/code English |
| Open source, original work | ✓ MIT, written from scratch for this project |
| KYC before payout | only if it wins; at payout stage |
