# Gnome Tavern — Quest Board test results

Run 2026-11-27, after Aldiyawan approved fetching the official Gno source
(phase-1 blocker in RESEARCH.md §2 is now resolved).

## Environment

- Toolchain: `gno v1.8.0` (prebuilt binaries in `tools/bin/`, as installed in phase 1).
- Source tree: **official** `gnolang/gno` release **v1.8.0**, downloaded as the
  GitHub tag archive `https://github.com/gnolang/gno/archive/refs/tags/v1.8.0.tar.gz`
  (51,585,497 bytes, SHA-256 `8c02780c3978303bb52c01bd93e0f4f112c990eb73a07de688a9ed0403a81b74`),
  extracted to `tools/gno-src-v1.8.0/`. The flagged phase-1 sparse clone in
  `tools/gno-src/` was NOT used.
- `GNOROOT=$HOME/workspace/gnome-tavern/tools/gno-src-v1.8.0` (verified via `gno env`).

## Commands run

```sh
export GNOROOT=$HOME/workspace/gnome-tavern/tools/gno-src-v1.8.0
export PATH=$HOME/workspace/gnome-tavern/tools/bin:$PATH
cd questboard
gno test -v .        # unit tests + filetests
gno lint .           # exit 0, no findings
gno fmt -diff .      # v1.8.0 has no -check flag; -diff is the check mode
gno fmt -w .         # applied the one formatting fix below
gno test -v .        # re-run after formatting: still green
```

## Results: 16/16 PASS, 0 test failures

Unit tests (`questboard_test.gno`) — 14/14 PASS:

| Test | Result |
|---|---|
| TestPostQuestHappy | PASS |
| TestPostQuestPaymentMustMatchExactly | PASS |
| TestPostQuestValidation | PASS |
| TestAcceptQuest | PASS |
| TestAcceptQuestGuards | PASS |
| TestCompleteQuestPaysAdventurer | PASS |
| TestCompleteQuestGuards | PASS |
| TestCancelQuestRefundsPoster | PASS |
| TestCancelQuestGuards | PASS |
| TestClaimRefundAfterDeadline | PASS |
| TestEscrowInvariantAcrossLifecycle | PASS |
| TestRenderPages | PASS |
| TestRenderSanitizesUserText | PASS |
| TestFormatGNOT | PASS |

Filetests (`questboard/filetests/`, picked up automatically by `gno test`) — 2/2 PASS:

| Filetest | Scenario | Result |
|---|---|---|
| z_0_filetest.gno | post → accept → complete (escrow pays adventurer) | PASS (gas 1,796,988) |
| z_1_filetest.gno | post → cancel (escrow refunds poster) | PASS (gas 1,492,719) |

Package verdict line from the tool: `ok .` — the realm package itself is green.

## Code changes made

Exactly one, formatting only:

- `questboard/questboard.gno` — `gno fmt -w` realigned the trailing comments
  in the validation-bounds const block (whitespace alignment). No logic,
  API, or design changes were needed: the phase-1 code compiled and passed
  as written, with the phase-1 security choices intact (payment via
  `cur.Previous().IsUserCall()` + `unsafe.OriginSend()`, per-call banker,
  checks-effects-interactions, stdlib-only).

## One non-code caveat: the "1 build error" line

`gno test` exits 1 and prints
`FAIL ./../../../.config/gno/pkg/mod/gno.land/r/gnometavern/questboard [setup failed]`
alongside the green `ok .` result, with the summary
`FAIL: 1 build errors, 0 test errors`. This is **not** a defect in the realm:

- In addition to the local package, the tool tries to fetch the module path
  `gno.land/r/gnometavern/questboard` from the mainnet RPC
  (`https://rpc.gno.land:443`) into the module cache and set that copy up as
  a second test target. The package is not deployed anywhere yet, and in
  this sandbox the Go client's TLS handshake to that RPC fails through the
  egress proxy (`tls: first record does not look like a TLS handshake`;
  plain `curl` to the same host succeeds, so it is a sandbox/proxy quirk of
  the tool's fetcher, not a network outage).
- All 16 tests that exercise our code pass; the failed target is the
  unreachable remote copy, not the local package.
- `gno lint .` shows the same harmless "downloading …" line and exits 0.

This does not block phase 2: deployment publishes the package on Onyx via
`gnokey` (RESEARCH.md §8), and local verification is complete.

## Verdict

**READY for phase 2** (Onyx testnet deployment per RESEARCH.md §8):
code compiles, 14/14 unit tests + 2/2 filetests pass, lint clean,
formatting clean. Remaining phase-2 steps are the ones already planned:
create the `tavern` gnokey, fund it from faucet.gno.land (Onyx), switch
`gnomod.toml` + `render.gno` RealmPath to the address-based namespace,
`gnokey maketx addpkg`, then verify in gnoweb and seed a quest.
