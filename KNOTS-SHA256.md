# What this fork changes

A fork of [Retropex/umbrel-bitcoin](https://github.com/Retropex/umbrel-bitcoin) (itself a fork of
[getumbrel/umbrel-bitcoin](https://github.com/getumbrel/umbrel-bitcoin)) that runs
[Bitcoin Knots SHA256](https://github.com/paulscode/knots-sha256): Bitcoin Knots' policy patch series maintained on
top of Bitcoin Core for the SHA256 chain, consensus-identical to Bitcoin Core. It backs two apps in
[paulscode/umbrel-store](https://github.com/paulscode/umbrel-store): the full node `paulscode-bitcoin-knots-sha256`
and the pruned companion `paulscode-knots-sha256`, which pin the same image.

Kept deliberately small so upstream's fixes keep applying:

1. **One bitcoind, the signed release.** `apps/backend/Dockerfile` downloads the published Guix build for the target
   architecture and verifies `SHA256SUMS.asc` against the maintainer's key (`apps/backend/knots-sha256-maintainer.asc`,
   fingerprint FF76 D484 3EBD 7FA0 6D92 DC0C B8AB 7B8E 7E28 0E1A) before using it; the build fails otherwise. The
   binaries are byte-identical to the release (they need only glibc, so no extra libraries). Upstream's twelve
   Knots 29.x builds are gone: they are other chains or older series. The version picker keeps one entry, and
   `AVAILABLE_BITCOIN_KNOTS_VERSIONS` must equal the directory under `/opt/bitcoind`.
2. **Stored versions from upstream** (Knots 29.x choices, "BIP110") are moved to "latest" at startup
   (`migrateBip110`), because the bitcoind manager refuses to start a version that is not installed.
3. **Settings follow Bitcoin Knots SHA256 31.1.** Every setting is written into `bitcoin.conf`, so the UI defaults
   override the node's: `blockmaxsize`/`blockmaxweight` default to the node's 300000 / 1200000 (upstream 3985000);
   `datacarriersize` stays 42 (no per-version override). `maxorphantx` is gone (removed by Bitcoin Core 31). The
   BLAKE2b blocknotify toggle is gone; the `datum` toggle stays (this is the chain the official Datum app mines).
   Other UI defaults that differ from bitcoind's (block filter index, txindex, buffers, dbcache, rpcworkqueue) are
   upstream's Umbrel choices and unchanged.
4. **Version banner** keeps the `.knots-sha256.<date>` suffix.
5. **Node 24** (Node 20 is end of life), pinned by digest.
6. From the BLAKE2b fork: pruning forces the block filter index off (a pruned node cannot build it and would stop),
   bounded bitcoind shutdown, `*.tsbuildinfo` kept out of the build context.

Build: `docker buildx build --platform linux/amd64,linux/arm64 --file apps/backend/Dockerfile -t
paulscode/umbrel-bitcoin-knots-sha256:<ver> --push .` (the store's `build/knots-sha256/build-umbrel-images.sh`).
