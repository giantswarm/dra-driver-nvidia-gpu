# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Giant Swarm packaging on top of the upstream source tree: `.circleci/config.yml` (app-catalog
  publishing), `.abs/` (app-build-suite config), `.kube-linter.yaml`, `CODEOWNERS`, this changelog.
- `icon` and the `application.giantswarm.io/team` / `io.giantswarm.application.audience`
  annotations in `Chart.yaml`, plus a `application.giantswarm.io/team` common label, so the
  chart shows up correctly in the customer-facing Backstage catalog.
- Chart moved from `deployments/helm/dra-driver-nvidia-gpu/` to `helm/dra-driver-nvidia-gpu/`
  to match the Giant Swarm `template-app` layout; `Makefile`, `hack/` and `demo/` paths follow.

### Changed

- Chart version is now decoupled from `appVersion` and continues the Giant Swarm line at
  **26.0.0**, while `appVersion` tracks upstream at `0.5.0`. Upstream re-versioned from
  calendar versions to semver, but this chart is published at `25.3.2-flatcar.1`, and under
  semver `0.5.0 < 25.3.2` — following upstream's number would make the catalog treat this as
  older than what clusters already run and never offer it as an upgrade. `26.0.0` rather than
  `25.4.0` because the release is breaking (see Removed, and the driver-570 requirement below).
  The image tag still derives from `appVersion`, so it resolves to `v0.5.0` as intended.

- Sync with upstream `kubernetes-sigs/dra-driver-nvidia-gpu` **v0.5.0** (was v25.3.2). Upstream
  moved from `NVIDIA/k8s-dra-driver-gpu` to `kubernetes-sigs/`, then re-versioned from calendar
  versions to semver — v25.12.0 was followed by v0.4.0, v0.4.1 and v0.5.0, so v0.5.0 is the
  latest release despite the lower number. Chart and appVersion track upstream exactly
  (`0.5.0`) and `kubeVersion: ">=1.32.0-0"` covers the certified Kubernetes 1.35.
- Default both `kubeletPlugin.networkPolicy.enabled` and `controller.networkPolicy.enabled` to
  `true` (upstream defaults to `false`). Giant Swarm clusters run a `default-deny-all`
  NetworkPolicy in `kube-system`, so with the upstream default the kubelet plugin's egress to
  the API server is dropped: the DaemonSet reports healthy, registers with kubelet, and never
  publishes a single `ResourceSlice`, so no GPU can ever be allocated through DRA. Verified on
  a Kubernetes 1.35 workload cluster with 4 × Tesla T4: without the policy
  `kubectl get resourceslices` stays empty and the plugin logs
  `dial tcp <apiserver>:443: i/o timeout`; with it the slice appears within seconds and a
  `ResourceClaim`-backed CUDA workload runs to completion.
- `image.repository` defaults to `gsoci.azurecr.io/giantswarm/k8s-dra-driver-gpu`, the Giant
  Swarm mirror of upstream's image — which moved in v0.5.0 from
  `nvcr.io/nvidia/k8s-dra-driver-gpu` to `registry.k8s.io/dra-driver-nvidia/dra-driver-nvidia-gpu`.
  The mirror must carry the `v0.5.0` tag before this chart can install from its defaults.
- CI: bump the `architect` orb from 8.0.2 to 10.5.0. The 8.x `push-helm` command defaults
  `registry_url` to the decommissioned `giantswarmpublic.azurecr.io`, so chart pushes failed
  with `dial tcp: lookup giantswarmpublic.azurecr.io: no such host`. From 9.0.0 the orb selects
  `gsoci.azurecr.io` or `gsociprivate.azurecr.io` based on repository visibility.
- CI: give app-build-suite's `HelmTemplateValidator` an extra values file
  (`.abs/helm-template-values.yaml`). The upstream chart deliberately refuses to render with
  defaults — `templates/validation.yaml` fails in the `default` namespace, and while
  `resources.gpus.enabled` is true without `gpuResourcesEnabledOverride` — and abs 2.3.0
  (pulled in by the orb bump) renders the chart as a build step.
- CI: set `override_app_version: false` on the catalog job. The chart derives its image tag from
  `appVersion` (`image.tag` defaults to `v<appVersion>`), so letting app-build-suite stamp the
  build version made dev builds request an image tag that was never built.

### Removed

- Drop five workflows inherited from the upstream tree that only make sense in upstream's
  repository: `stale.yml` and `issue-triage.yml` (upstream's bots, which would start labelling
  and closing Giant Swarm issues), `release-automation.yml` (cuts releases from a `VERSION`
  file — Giant Swarm releases via CircleCI and the architect orb), `mock-nvml-e2e.yaml` (runs
  on paths like `deployments/**` and `tests/**` that this fork no longer has) and `tests.yaml`
  (a stub whose only job reports that bats runs on upstream's Prow cluster). `helm.yaml` is
  kept — `basic-checks.yaml` calls it, and its `make helm-lint` target already follows the
  chart to `helm/`.

- Drop the downstream Flatcar patch series: upstream now searches `/opt/bin` for `nvidia-smi` in
  both `cmd/*/root.go` and `hack/kubelet-plugin-prestart.sh` and follows symlinks when resolving
  `libnvidia-ml.so.1`, which is exactly what the Giant Swarm build added. The
  `EXTRA_DRIVER_BINARY_PATHS`/`EXTRA_DRIVER_LIBRARY_PATHS` handling and the
  `extraDriverBinaryPaths`, `extraDriverLibraryPaths` and `extraHostPathMounts` values are gone
  with it, so the chart no longer carries the `-flatcar.N` pre-release suffix and no longer needs
  a patched image. **Breaking** for anyone setting those three values.
- Drop the upstream `tests/bats` suite, which targets NVIDIA's own multi-node NVLink test rigs.

### Upgrade notes

- **Requires NVIDIA driver 570 or newer.** Flatcar 4593.2.5 defaults to 535.274.02, on which the
  kubelet plugin's init container loops forever: its prestart script runs `nvidia-smi --version`,
  which 535's `nvidia-smi` rejects with `Option --version is not recognized`. Flatcar 4593 also
  ships 570.195.03, selectable per node by writing `NVIDIA_DRIVER_VERSION=570.195.03` to
  `/etc/flatcar/nvidia-metadata`, which `/usr/lib/nvidia/bin/setup-nvidia` sources after the
  shipped default. (Driver 535 additionally trips the v25.12.0-era
  `undefined symbol: nvmlDeviceGetAddressingMode` crash; upstream made that symbol optional in
  v0.4.0, so v0.5.0 gets past it and fails at the `nvidia-smi --version` check instead.)
- **GPU allocation needs two values.** `resources.gpus.enabled` is true by default, but
  `templates/validation.yaml` hard-fails unless `gpuResourcesEnabledOverride: true` is also set,
  because the driver must not run alongside the standard GPU device plugin until KEP 5004 is GA.
  Disable the GPU operator's device plugin on any node where this driver announces GPUs.
- **Disable compute domains on non-NVLink hardware.** `resources.computeDomains.enabled` defaults
  to true and the `compute-domains` container needs the `nvidia-caps-imex-channels` character
  device (NVLink-fabric/GB200-class). On e.g. `g4dn` (Tesla T4) the module is absent and the
  container crashes at startup with `error getting nvcap for IMEX channel '0': ... error parsing
  '/proc/devices': unexpected regex match: []`.
- `kubeletPlugin.tolerations` now defaults to `nvidia.com/gpu: Exists:NoSchedule` upstream, so
  GPU node pools using that taint no longer need it set explicitly.

