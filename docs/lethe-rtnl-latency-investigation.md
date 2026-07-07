# Lethe rtnl latency investigation

This branch captures the live-tested AWS VPC CNI changes from the Lethe EKS
pod-create latency investigation on 2026-07-07.

## What was tried

1. Removed the per-pod host `LinkList()` MAC uniqueness scan from `setupVeth`.
   This replaced random MAC generation plus full host-link dump with a stable
   locally administered unicast MAC derived from `hostVethName|podIP`.
2. Added temporary `LETHE_TIMING` debug logs around the veth, route, address,
   neighbor, and host route/rule netlink operations.
3. Changed normal veth creation to create the host-side veth in the host
   namespace and place the peer directly in the pod namespace with
   `netlink.Veth.PeerNamespace = netlink.NsFd(podNS.Fd())`. This avoids the old
   create-in-pod-netns then `LinkSetNsFd` host-veth move.

## Live EKS result

The MAC-only patch was insufficient:

| run | CNI p50 | post-IPAM plugin/veth p50 |
| --- | ---: | ---: |
| MAC-only unpaced | 4.289s | 3.108s |
| MAC-only 150ms paced | 0.262s | 0.125s |

The diagnostic image showed the old path queueing behind rtnl across multiple
operations, especially:

| old-path operation | p50 | p90 | max |
| --- | ---: | ---: | ---: |
| `host.WithNetNSPath` | 1463ms | 2594ms | 2764ms |
| `container.LinkSetNsFdHost` | 484ms | 889ms | 1029ms |
| `container.NeighAdd` | 378ms | 759ms | 1029ms |
| `host.LinkByNameHost` | 231ms | 879ms | 920ms |

The `PeerNamespace` candidate materially reduced the hot path:

| run | RunPodSandbox p50 | CNI p50 | post-IPAM plugin/veth p50 |
| --- | ---: | ---: | ---: |
| `peer-ns-rc-unpaced` | 1.667s | 0.717s | 0.0285s |
| `peer-ns-rc-150ms` | 1.217s | 0.141s | 0.006s |

The cluster was reverted after testing to the managed EKS add-on image and
config:

- `amazon-k8s-cni:v1.21.2-eksbuild.2`
- `enableNetworkPolicy=true`
- `ENABLE_PREFIX_DELEGATION=true`
- `WARM_PREFIX_TARGET=2`

## Validation caveats

This is a live-tested investigation branch, not a polished upstream PR yet.

- The `LETHE_TIMING` logs should be removed or gated before production use.
- Existing driver mocks still need to be updated for the new netlink call
  sequence.
- `go test -run '^$' ./cmd/routed-eni-cni-plugin/driver` passes as a compile
  check.
- Full `go test ./cmd/routed-eni-cni-plugin/driver` currently fails because
  tests expect the old `LinkAdd` and `LinkSetNsFd` sequence.
- Broader validation is still needed for branch ENI, IPv6, and rollback/error
  paths.
