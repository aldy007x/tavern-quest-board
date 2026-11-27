# Gnome Tavern — Phase 2a results (deploy + repo)

Run 2026-11-27, after Aldiyawan approved the full phase 2
("Gas fase 2 Gnome Tavern sampai submit"). This file covers deploy + repo;
the demo video and the DoraHacks submission are separate phase-2 steps.

## Network

- Gno.land **Onyx testnet** (Test17), chain ID `onyx-1`
- RPC: `https://rpc.onyx.testnets.gno.land:443`
- Realm path: `gno.land/r/g13fd5s4h9ap0cggkgxnkqf0wawm9ew8x2kvnlv5/questboard`
- gnoweb URL: https://latest.testnets.gno.land/r/g13fd5s4h9ap0cggkgxnkqf0wawm9ew8x2kvnlv5/questboard

## Wallets (testnet only)

Keybase: `tools/keybase/` (gnokey `-home`). Mnemonics are stored ONLY in
`tools/WALLET-BACKUP.txt` (mode 600, excluded from git) — never in this file.

| Key | Role | Address |
|---|---|---|
| `tavern` | deployer / quest poster | `g13fd5s4h9ap0cggkgxnkqf0wawm9ew8x2kvnlv5` |
| `adventurer` | demo counterparty | `g1rvu4mjutxcumkrwpld30t35xmhh47rtcqwmscj` |

The `tavern` address is also the GNOT prize-payout address for the
DoraHacks form (RESEARCH.md §8).

## Funding

The web faucet (`faucet.onyx.testnets.gno.land`, JSON-RPC `drip`) requires a
reCAPTCHA token, so it cannot be driven from this sandbox. Funding was done
through the **official agent faucet** for Onyx instead
(`POST https://faucet-agent.onyx.testnets.gno.land/fund`), 50 GNOT each:

| Address | Amount | Drip tx hash |
|---|---|---|
| tavern | 50,000,000 ugnot | `f66fcc8f4d116b4c32455608845854e3ff8a121fa80be2f43a3827f331df7d4e` |
| adventurer | 50,000,000 ugnot | `99c3877e537e1ee5ac80f018a285a2f12faefb15ab310fc29c031c045591b404` |

Balances verified on chain via `gnokey query bank/balances/...`:
both accounts showed `"50000000ugnot"`.

### Sandbox RPC relay (why gnokey points at 127.0.0.1:26657)

`gnokey` ignores `HTTP(S)_PROXY` env vars, and its direct TLS connections to
the Onyx RPC are intercepted in this sandbox ("tls: first record does not
look like a TLS handshake" — the same quirk documented in TEST-RESULTS.md).
All `gnokey` chain traffic therefore goes through a small local relay,
`tools/rpc-relay.py`, which listens on `http://127.0.0.1:26657` and forwards
to the real RPC over HTTPS via the sandbox egress proxy. Start it with:

```sh
python3 tools/rpc-relay.py   # RELAY_UPSTREAM defaults to the Onyx RPC
```

and use `-remote http://127.0.0.1:26657` on gnokey commands.

## Namespace switch + retest

- `questboard/gnomod.toml`: module →
  `gno.land/r/g13fd5s4h9ap0cggkgxnkqf0wawm9ew8x2kvnlv5/questboard`
- `questboard/render.gno`: `RealmPath` const and the in-page `/r/...` links →
  the same address-based path; filetest imports updated to match.
- Re-ran `gno test .`: local package verdict **`ok .`** (all 16 tests pass,
  0 test errors). The single "build error" line is the known non-defect from
  TEST-RESULTS.md (the tool trying to fetch the not-yet-deployed remote copy
  of the module from mainnet RPC).

## Deploy

First attempt with the RESEARCH.md §8 gas values (wanted 10,000,000) failed
in simulation: the realm needs **40,912,806 gas**. Retried with
`-gas-wanted 45000000 -gas-fee 100000ugnot -max-deposit 2000000ugnot`:

- **Deploy tx hash: `VOErgPztlvTd5M2ltWv6YsWyRuf6UlIH5ZvcEwDB31k=`**
- Height: **1439555**, result `success: true`, gas used 40,912,806
- PKGPATH: `gno.land/r/g13fd5s4h9ap0cggkgxnkqf0wawm9ew8x2kvnlv5/questboard`

### Enablement (GPAS) — the one external wait

Onyx runs the mainnet-line two-phase deploy: `MsgAddPackage` **parks** the
package (`code_submission_policy = "inert"`) and it only goes live when an
approver sends `MsgEnablePackage` after the GPAS checks pass.

- GPAS verdict for this package: **passed** at 2026-11-27T01:23:27Z
  (https://gpas.latest.testnets.gno.land/?path=gno.land/r/g13fd5s4h9ap0cggkgxnkqf0wawm9ew8x2kvnlv5/questboard)
- Until the enable lands, `vm/qrender` answers "package not found" and
  gnoweb returns 404 — expected behaviour for a parked package, not a
  deploy failure. Other packages passed in the same window
  (`.../bench` 01:14Z, `.../verification` 00:20Z) were still parked at last
  check, so the approver is batching/lagging, not singling this package out.

## Live lifecycle demo (PostQuest → AcceptQuest → CompleteQuest)

_Pending package enablement — see above. Tx hashes and before/after
balances will be recorded here once the realm is callable._

## GitHub repo

- **https://github.com/aldy007x/tavern-quest-board** (PUBLIC)
- Contents: `questboard/` (realm code, tests, filetests, gnomod.toml),
  `README.md`, `RESEARCH.md`, `TEST-RESULTS.md`, `LICENSE`, `.gitignore`.
- `tools/` is git-ignored: no toolchain binaries, no keybase, and
  **no wallet backup** in the repo.
