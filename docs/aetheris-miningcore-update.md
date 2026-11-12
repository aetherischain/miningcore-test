# Aetheris Miningcore compatibility patch

Aetheris source confirms Scrypt PoW and Bitcoin-style GBT, but its serialization is not identical to the generic Miningcore Bitcoin job. This patch adds a coin-specific job selected by symbol `AETH`:

- includes the timestamp field in the coinbase transaction;
- appends the serialized empty PoW block-signature vector (`00`);
- limits submitted header time to the currently active Aetheris 15-second future-drift window relative to the template time.

The existing candidate coin template already specifies SHA256d coinbase hashing, Scrypt `N=1024, r=1` header hashing, and reverse-SHA256d block ID hashing for the observed block version. This patch does not change those parameters.

## Apply on the Linux checkout

From the root of the outer `aetherischain/miningcore-test` checkout, on the update branch:

```bash
git switch copilot/aetheris-miningcore-support
git apply --check patches/aetheris-miningcore.patch
git apply patches/aetheris-miningcore.patch
```

Then build from the vendored Miningcore source using its documented build procedure. Do not copy the resulting binary into the live service yet. Keep Aetheris pool payments disabled, Stratum/API bound to loopback, and do not submit blocks or expose port 3333 during initial validation.

## Validation still required

This is a compatibility implementation candidate, not a production certification. The repository automation environment did not compile or run this C# patch. Before launch:

1. Review the patch against the exact vendored source revision; `git apply --check` must pass.
2. Build Miningcore and run its Bitcoin-family test suite.
3. Add/run serialization tests against Aetheris-generated PoW block and coinbase bytes, including merkle root, header hash, and block ID; compare with daemon source or a local test fixture.
4. Check the `AETH` address's decoded pubkey hash and generated scriptPubKey locally against Aetheris `validateaddress`/`getaddressinfo`. Do not infer this only from the Base58 prefix.
5. Recheck Aetheris `FutureDrift` consensus constants before deployment; update the 15-second bound if the active rules differ.
6. Only after these pass, run an isolated local Stratum test with payments disabled. Do not enable payouts until a controlled block-submission test is accepted and accounting is verified.

The patch is a source change, not a command to start the pool. Do not run Miningcore before the validation steps above.
