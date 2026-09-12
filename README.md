# zoorikctl

The command a customer runs to connect a Kubernetes cluster to Zoorik. Binaries are published on
this repository's Releases page; the source lives in `storagetax/storagetax-k8s-optimizer`
under `cli/zoorikctl`.

| | |
|---|---|
| macOS | `brew tap storagetax/tap && brew install zoorikctl` or `curl -fsSL https://get.zoorik.com/macos \| sh` |
| Linux | `curl -fsSL https://get.zoorik.com/linux \| sh` |
| Windows | `scoop bucket add storagetax https://github.com/storagetax/scoop-bucket && scoop install zoorikctl` or `irm https://get.zoorik.com/windows \| iex` |

Every installer verifies the archive against the release's signed `checksums.txt`.
