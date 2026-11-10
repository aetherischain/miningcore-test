# Miningcore test setup — Aetheris

> Internal setup notes. This file intentionally excludes RPC credentials, PostgreSQL passwords, private keys, and seed phrases. A GitHub private repository is still accessible to its collaborators; do not add secrets here.

## Goal and current status

Test the `theretromike/miningcore` Docker image for an Aetheris pool while keeping the existing NOMP service running. The test must begin with payment processing disabled and must not expose the Stratum port publicly until compatibility is verified.

**Status:** PostgreSQL and the Miningcore image are installed locally. Miningcore has not been confirmed running. Do not start it using the earlier suggested `/etc/miningcore:/app` bind mount: mounting over `/app` may hide the executable and bundled files. The earlier sample Aetheris coin definition is provisional and must not be treated as validated.

## Server facts reported in the setup session

- OS: Ubuntu 22.04 (reported)
- Existing NOMP container: `rmt-nomp`, image `theretromike/rmt-nomp:latest`; it was left running.
- Miningcore image pulled: `theretromike/miningcore:latest`
- Miningcore image digest: `sha256:23543c587cc9899c046ced3dd2f472bcae09610dedc0a8f7b6a24c25de039727`
- Miningcore image command: `/app/build/Miningcore -c /app/config.json`
- PostgreSQL container: `miningcore-postgres`, image `postgres:16`, bound to `127.0.0.1:5432` only.
- PostgreSQL database/user: `miningcore` / `miningcore` (password deliberately not recorded).
- Aetheris daemon runs on the host as `aetherisd`.
- Aetheris RPC listens on `127.0.0.1:25197`; P2P listens on `25198`.
- Aetheris config path: `/root/.aetheris/aetheris.conf`; permissions were changed to `600`.
- RPC user and password are configured in that file; values deliberately not recorded.
- Proposed Stratum port: `3333` (do not open it to the Internet until testing is complete).
- Candidate pool payout address supplied and validated by the daemon: `AcK5HfAWFQJqHjiePGnExZ5DkUckmr6j2G`. It is public, but consider using a dedicated pool wallet/address.

## Aetheris data observed

`getblockchaininfo` reported:

- chain `main`
- height around `71800`, fully synchronized (`blocks == headers`)
- both `proof-of-work` and `proof-of-stake` difficulty fields

`getblocktemplate` returned a Bitcoin-style GBT object with height around `71801`, `version` 536870912 (`0x20000000`), `bits` `1b0cccbc`, `target` present, and `coinbasevalue` 12000000000 base units. The daemon's source indicates PoW Scrypt (N=1024, r=1, p=1), a PoW cutoff at mainnet height 7,000,000, and a distinct block-ID hash path for version > 6. These observations make a generic Scrypt coin template only a hypothesis; they do not prove Miningcore can hash shares, serialize blocks, or handle pool payouts correctly.

## Important corrections / safeguards

- Do not set `isPseudoPoS: true` for the initial Aetheris PoW test. This can affect Miningcore block serialization and address handling.
- Do not assume a `getblocktemplate` extra argument such as `"scrypt"` is accepted by Aetheris; the observed GBT call succeeded without it. Confirm the daemon's RPC implementation before adding arguments.
- Do not enable cluster-level or pool-level payment processing during compatibility testing.
- Do not publish the daemon RPC port or credentials. Prefer host networking only if required for a container to reach the host RPC, and keep RPC bound to loopback. Restrict any additional API/Stratum listeners with firewall rules until tested.
- Never commit `aetheris.conf`, `config.json` containing real credentials, wallet files, private keys, or recovery material.
- Do not stop or remove NOMP as part of this test.

## Next steps

1. Inspect `/etc/miningcore/config.json` locally with a script that reports only non-secret fields; do not print the whole file.
2. Verify the Miningcore image's bundled paths and coin-template loader, and inspect its default `coins.json` without overwriting the image's `/app` directory.
3. Create a separate host directory for test configuration and mount only the specific config and coin-template files at verified container paths.
4. Add a provisional Aetheris Bitcoin-family coin definition with PoW Scrypt and the matching block-hash configuration; validate all hash byte order and payout-address/network handling against source and tests before launch.
5. Keep payment processing disabled; start Miningcore in a restricted test mode and review logs for daemon startup, `getblocktemplate`, and Stratum initialization.
6. Test with a controlled local miner or test harness first. Do not submit a production block or enable payout processing until block serialization and payout accounting are verified.

## Useful local checks

```bash
docker ps -a --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}'
docker image inspect theretromike/miningcore --format 'ID={{.Id}} Cmd={{json .Config.Cmd}}'
ss -ltnp | grep -E ':(25197|25198|3333|4000)\b' || true
aetheris-cli getblockchaininfo
aetheris-cli getblocktemplate
```
