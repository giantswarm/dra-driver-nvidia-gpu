# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Initial Giant Swarm fork: vendir-based upstream sync from `NVIDIA/k8s-dra-driver-gpu`.
- `sync/patches/team-label`: injects `application.giantswarm.io/team` into the chart's common labels.
- `sync/patches/network-policies`: explicit ingress deny + DNS egress on both upstream NetworkPolicies.
- `sync/patches/kyverno-policies`: Kyverno `PolicyException` template for the PSS-style policies that would otherwise block the driver's privileged + hostPath workloads (off by default; enable via `.Values.kyvernoPolicyExceptions.enabled`).
- Chart moved from `deployments/helm/dra-driver-nvidia-gpu/` to `helm/dra-driver-nvidia-gpu/` to match the Giant Swarm `template-app` layout.
- `.circleci/config.yml`, `values.schema.json` for app-catalog publishing.
- `icon` field in `Chart.yaml` pointing to `https://s.giantswarm.io/app-icons/kubernetes-gpu/1/light.svg` so the chart renders an icon in the customer-facing Backstage catalog.

### Changed

- Sync with upstream `kubernetes-sigs/dra-driver-nvidia-gpu` v25.12.0 (was v25.3.2); upstream moved there from `NVIDIA/k8s-dra-driver-gpu`. Chart and appVersion now track upstream exactly (`25.12.0`), and `kubeVersion: >=1.32.0-0` covers the certified Kubernetes 1.35.

### Removed

- Drop the downstream Flatcar patch series: upstream v25.12.0 now searches `/opt/bin` for `nvidia-smi` in both `cmd/*/root.go` and `hack/kubelet-plugin-prestart.sh` and follows symlinks when resolving `libnvidia-ml.so.1`, which is exactly what the Giant Swarm build added. The `EXTRA_DRIVER_BINARY_PATHS`/`EXTRA_DRIVER_LIBRARY_PATHS` handling and the `extraDriverBinaryPaths`, `extraDriverLibraryPaths` and `extraHostPathMounts` values are gone with it, so the chart no longer carries the `-flatcar.N` pre-release suffix and no longer needs a patched image. **Breaking** for anyone setting those three values.
