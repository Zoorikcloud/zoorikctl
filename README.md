# zoorikctl

The command a customer runs to connect a Kubernetes cluster to Zoorik. Binaries are published on
this repository's Releases page; the source lives in `Zoorikcloud/k8spilot`
under `cli/zoorikctl`.

| | |
|---|---|
| macOS | `brew tap zoorikcloud/tap && brew trust zoorikcloud/tap && brew install zoorikctl` or `curl -fsSL https://get.zoorik.com/macos \| sh` |
| Linux | `curl -fsSL https://get.zoorik.com/linux \| sh` |
| Windows | `scoop bucket add zoorikcloud https://github.com/Zoorikcloud/scoop-bucket && scoop install zoorikctl` or `irm https://get.zoorik.com/windows \| iex` |

Every installer verifies the archive against the release's `checksums.txt`, and against that
file's cosign signature when the release publishes one. v0.2.0 was built on a maintainer's
machine and carries no signature; releases from the build workflow are signed.
