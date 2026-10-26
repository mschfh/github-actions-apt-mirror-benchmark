# github-actions-apt-mirror-benchmark

Benchmarks APT mirrors on GitHub-hosted runners (`ubuntu-latest` amd64 + `ubuntu-24.04-arm` arm64) to track the
`azure.archive.ubuntu.com` "priority 1" slowness problem and find faster alternatives.

Run it: **Actions → apt-mirror-benchmark → Run workflow** (or wait for the daily cron).

## What it does (read-only, nothing is reconfigured)

1. Dumps the runner's actual mirror config: `/etc/apt/apt-mirrors.txt` (the
   `mirror+file:` failover list that apt resolves: azure p1 → archive p2 → security p3).
2. **Baseline**: sequential `curl` of `ls-lR.gz` (~37 MB) from *every* `apt-mirrors.txt` entry
   (azure, archive, security; ARM ports equivalents where configured) — metrics: http code,
   TTFB, bytes, MB/s.
3. **Top-10 US sweep** via [`vegardit/fast-apt-mirror.sh`](https://github.com/vegardit/fast-apt-mirror.sh)
   (pinned to commit `806c5927…` = tag `1.4.5`): health-probes US mirrors from `mirrors.ubuntu.com/US.txt`, then speed tests
   the top 10 + the current mirror + the archive reference, **strictly sequential**
   (`--parallel 1`), full-file ~37 MB sample, 30 s cap per mirror (`--sample-time 30`).
   On arm64 the tool swaps candidates to the `/ubuntu-ports` tree automatically.
   `find` without `--apply` never touches `sources.list`.
4. Merged ranked table → workflow run summary; raw logs as artifacts (90 days).

## Notes

- There is no `ubuntu-latest-arm` label (yet); the matrix uses `ubuntu-24.04-arm`.
- The sample file is `ls-lR.gz` intentionally: identical byte count on the `/ubuntu` and
  `/ubuntu-ports` trees, so amd64 and arm64 results are directly comparable. Note that a real
  `apt-get update` mostly pulls `Contents-<arch>.gz` (~49 MB) instead — same ballpark.
- Health/HEAD probes run concurrently inside `fast-apt-mirror.sh` (configurable only via its
  upstream code path); all *throughput* tests are sequential, per the tool's `--parallel 1`.
- `InRelease` files are tiny (~120–250 KB) and only measure connection establishment — which is
  why this benchmark downloads a full 37 MB file instead.
