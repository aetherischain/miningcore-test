# Aetheris / Miningcore compatibility test package

This folder contains a **candidate, not validated or production-ready** Miningcore setup. It is intentionally not safe to start until the blockers below are resolved. It contains no RPC or PostgreSQL credentials.

## What's prepared

- `coins.candidate.json`: AETH symbol and provisional Bitcoin-family PoW Scrypt settings.
- `config.candidate.json`: local-only API and Stratum listeners, daemon RPC at `127.0.0.1:25197`, cluster and pool payments disabled, and local credential placeholders.

The candidate coin definition uses SHA256d coinbase hashing, Scrypt (N=1024, r=1) header hashing, reversed SHA256d block hash, and shareMultiplier 65536 as a starting hypothesis. This is based on the Miningcore Scrypt coin entries and prior Aetheris node observations; it is **not evidence** that Aetheris block serialization, hybrid PoW/PoS behavior, payout scripts, or share validation are compatible.

## Blocking validation before launch

1. Compare Aetheris node source for its active-height consensus, PoW hash, block ID, coinbase rules, and `getblocktemplate`/`submitblock` behavior with Miningcore's `BitcoinJob` and `BitcoinJobManager` implementation.
2. Confirm how this Miningcore build loads external `coinTemplates` and whether the candidate config fields are accepted by the exact build/tag. Verify the config path in the image before mounting anything. Do **not** bind-mount a host directory over `/app`.
3. Address validation is a known risk: Miningcore's Bitcoin helper decodes Base58Check but does not, by itself, establish Aetheris network-prefix semantics. Verify the Aetheris address's scriptPubKey is built identically by Miningcore, and verify coinbase construction before using this address.
4. Check whether this chain's current height uses PoW or PoS and whether the single `headerHasher`/`blockHasher` fields can represent it. Do not copy `isPseudoPoS` or GBT arguments from another coin without source-level proof.
5. Keep `paymentProcessing.enabled` false and do not expose port 3333. The sample binds API/Stratum to loopback only. Do not test block submission or real payouts until consensus and serialization are proven.

## Relevant files in the vendored Miningcore source

- `vendor/rmt-miningcore/src/Miningcore/coins.json` — built-in coin hash definitions; compare Scrypt entries such as `digibyte-scrypt`.
- `vendor/rmt-miningcore/src/Miningcore/Blockchain/Bitcoin/BitcoinJob.cs` — coinbase, merkle, header/share hashing, and block construction.
- `vendor/rmt-miningcore/src/Miningcore/Blockchain/Bitcoin/BitcoinJobManager.cs` and `BitcoinJobManagerBase.cs` — GBT polling and job lifecycle.
- `vendor/rmt-miningcore/src/Miningcore/Blockchain/Bitcoin/BitcoinUtils.cs` — address-to-script conversion; inspect network prefix handling.
- `vendor/rmt-miningcore/src/Miningcore/Blockchain/Bitcoin/BitcoinPool.cs` — Bitcoin-family pool initialization.
- `vendor/rmt-miningcore/src/Miningcore/Blockchain/Bitcoin/Configuration/BitcoinPoolConfigExtra.cs` — `GBTArgs`, legacy daemon and address options.
- `vendor/rmt-miningcore/src/Miningcore/Configuration/ClusterConfig.cs` — pool/coin-template config model and coin-family registration.
- `vendor/rmt-miningcore/examples/digibyte_scrypt_pool.json` — example Scrypt pool config; credentials and network values there are examples only.

## Linux transfer and local setup

The repository is private; authenticate interactively through GitHub when prompted. Never put a PAT, RPC secret, or database password in a command, URL, shell history, or this repository.

```bash
sudo apt-get update && sudo apt-get install -y git
mkdir -p "$HOME/src"
cd "$HOME/src"
git clone https://github.com/aetherischain/miningcore-test.git
sudo install -d -m 700 /etc/miningcore-aetheris
sudo install -m 600 deploy/aetheris/coins.candidate.json /etc/miningcore-aetheris/coins.candidate.json
sudo install -m 600 deploy/aetheris/config.candidate.json /etc/miningcore-aetheris/config.json
```

Edit `/etc/miningcore-aetheris/config.json` locally to replace both RPC placeholders and both PostgreSQL placeholders. Do not paste the edited file into chat or commit it. The daemon configuration has been reported at `/root/.aetheris/aetheris.conf` with mode 600; obtain credentials locally without printing them. Confirm how Docker on this host reaches the node's loopback RPC before running a container; `127.0.0.1` inside a bridged container is the container itself. Do not change the daemon RPC bind to a public interface. If host networking is required, ensure port 3333 and API port 4000 remain firewalled and the listeners stay loopback-only.

Before any start command, inspect the exact image's embedded executable/config/coins paths and configuration-loading behavior. Do not mount the entire `/etc/miningcore-aetheris` directory onto `/app`. Keep the existing NOMP service untouched.

Validate JSON after local edits without exposing it:

```bash
python3 -m json.tool /etc/miningcore-aetheris/config.json >/dev/null && echo 'config JSON OK'
python3 -m json.tool /etc/miningcore-aetheris/coins.candidate.json >/dev/null && echo 'coin JSON OK'
stat -c '%a %n' /etc/miningcore-aetheris/config.json /etc/miningcore-aetheris/coins.candidate.json /root/.aetheris/aetheris.conf
```

Do not start Miningcore using these candidates until the blocking validation is complete. First launch, when approved, should be an isolated local test with payments disabled, no public listeners, and NOMP still running.
