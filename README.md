# 🍺 Tavern Quest Board

An on-chain bounty board for the **Gnome Tavern**, built as a [Gno](https://docs.gno.land) realm for [Gno.land](https://gno.land).

Post a quest with a reward in GNOT. The reward is escrowed by the realm the moment the quest is posted. Another user — the adventurer — accepts the quest and does the work. When the poster marks the quest complete, the escrowed reward is paid to the adventurer. Nobody completes it in time? The poster reclaims the full escrow after the deadline. No middleman, no trust required: the realm holds the gold and the rules are the code.

Built for the **Gnome Tavern Hackathon | Gno.land** on DoraHacks (submission deadline 18 Dec 2026).

## How it works

Quest lifecycle:

```
Open ──AcceptQuest──▶ Accepted ──CompleteQuest──▶ Completed (adventurer paid)
Open ──CancelQuest──▶ Cancelled (poster refunded)
Open|Accepted ──ClaimRefund, after deadline──▶ Expired (poster refunded)
```

Exported functions (callable via `gnokey` or the gnoweb **Actions** tab):

| Function | Who | What |
|---|---|---|
| `PostQuest(title, description, reward, deadline)` | anyone | Creates a quest. Attach exactly `reward` ugnot with `-send`; the realm verifies the payment against the transaction's send envelope and escrows it. |
| `AcceptQuest(id)` | anyone except the poster | Takes an open quest before its deadline. |
| `CompleteQuest(id)` | poster | Releases the full escrowed reward to the adventurer. |
| `CancelQuest(id)` | poster | Withdraws an open quest; full refund. |
| `ClaimRefund(id)` | poster | After the deadline, refunds an open or accepted quest in full. |
| `GetQuest(id)`, `QuestCount()`, `OpenQuestCount()`, `EscrowedTotal()` | anyone | Read-only queries. |

The realm also implements `Render(path)`, so the board is browsable on gnoweb: home (`""`), `open`, `all`, and `quest/<id>` pages, with user-supplied text sanitized before it is rendered as markdown.

### Design choices

- **Escrow with checks-effects-interactions.** State is settled before any coin moves; the realm's escrow accounting (`EscrowedTotal`) always equals the sum of rewards of open/accepted quests, and the tests assert this invariant across the lifecycle.
- **Canonical payment verification.** `PostQuest` requires a direct user call (`cur.Previous().IsUserCall()`) and reads the paid amount from the message send envelope — the pattern used by the official `r/demo/disperse` example — instead of trusting a parameter.
- **Standard library only.** No external `gno.land/p/...` dependencies, so the realm compiles and tests anywhere the Gno toolchain runs.
- **Testable core.** Crossing functions are thin wrappers: they extract caller, payment, time and a `chain/banker` handle, then delegate to internal functions that unit tests drive with an in-memory fake banker.

## Repository layout

```
questboard/
  gnomod.toml            module = "gno.land/r/g13fd5s4h9ap0cggkgxnkqf0wawm9ew8x2kvnlv5/questboard"
  questboard.gno         state, lifecycle logic, exported functions
  render.gno             gnoweb Render() + markdown sanitizing
  questboard_test.gno    unit tests (fake-banker state machine tests)
  filetests/             end-to-end filetests (simulated chain transactions)
RESEARCH.md              ecosystem research + Onyx deployment plan
```

## Run the tests

Requires the Gno toolchain (`gno`, `gnokey`, `gnodev`; see [RESEARCH.md](RESEARCH.md) for installation and the current toolchain status in our sandbox):

```sh
cd questboard
gno fmt ./...
gno lint ./...
gno test ./...        # unit tests + filetests
```

Local development with hot reload and a web UI:

```sh
gnodev .              # from the questboard/ directory; open http://localhost:8888
```

## Deploy to the Onyx testnet

Summary (full walkthrough with faucet and verification steps in [RESEARCH.md](RESEARCH.md)):

```sh
gnokey add tavern                                     # create a key, save the mnemonic
# fund the address at https://faucet.gno.land (network: Onyx)
gnokey maketx addpkg \
  -pkgpath "gno.land/r/<your-g1-address>/questboard" \
  -pkgdir . \
  -gas-fee 10000ugnot -gas-wanted 10000000 \
  -chainid onyx-1 -remote https://rpc.onyx.testnets.gno.land:443 \
  tavern
```

Then browse it at `https://latest.testnets.gno.land/r/<your-g1-address>/questboard`.

> Testnet only. Testnet GNOT has no value; never use a development mnemonic for real funds.

## License

MIT — see [LICENSE](LICENSE). Original work for the Gnome Tavern Hackathon.
