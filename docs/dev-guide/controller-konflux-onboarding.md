# Downstream controller Konflux onboarding

The RHOAI controller build is separate from the upstream ODH controller build.
Inherited PipelineRuns targeting `open-data-hub-tenant`, the upstream repository,
or `quay.io/opendatahub/` do not onboard a downstream release.

## Build identity for RHOAI 3.7 EA1

| Setting | Value |
| --- | --- |
| Source repository | `https://github.com/red-hat-data-services/data-connect-hub` |
| Release branch | `rhoai-3.7-ea.1` |
| Application | `rhoai-v3-7-ea-1` |
| Component | `odh-data-connect-hub-controller-v3-7-ea-1` |
| Namespace | `rhoai-tenant` |
| Build service account | `build-pipeline-odh-data-connect-hub-controller-v3-7-ea-1` |
| Output repository | `quay.io/rhoai/odh-data-connect-hub-controller-rhel9` |
| Containerfile | `dc-controller/Containerfile.konflux` |
| Build context | `.` (repository root) |
| Dependency prefetch | `[{"type": "gomod", "path": "dc-controller"}]` |

PR builds run on `stone-prod-p01` and publish `on-pr-{{pull_request_number}}`
images for x86_64 and arm64, expiring after 30 days. Release pushes run on
`stone-prod-p02` and publish `rhoai-3.7-ea.1` and branch/commit tags for x86_64,
arm64, ppc64le, and s390x. Both builds are hermetic and produce source images.
The controller uses the pipeline's blocking FIPS-check default; the Rust
services' FIPS exception must not be copied to it.

## Repository changes

Pipeline definitions are maintained in
[`red-hat-data-services/konflux-central`](https://github.com/red-hat-data-services/konflux-central),
not directly in this repository's `.tekton/` directory:

- On `main`, add
  `pipelineruns/data-connect-hub/.tekton/odh-data-connect-hub-controller-pull-request.yaml`.
  This follows the existing REST/Flight multi-branch PR routing.
- On `rhoai-3.7-ea.1`, add
  `pipelineruns/data-connect-hub/.tekton/odh-data-connect-hub-controller-v3-7-ea-1-push.yaml`.
- On `main`, add the controller image repository to the `data-connect-hub`
  entry in `config/component_repo_map.json` for grouped build testing.

In this source repository, both branches must load `/cachi2/cachi2.env` before
downloading Go modules and compiling. Without that environment the hermetic
build can attempt to access the network instead of the prefetched dependencies.
The conditional load preserves normal local builds without Konflux.

Keep `main`'s ODH image defaults. On the downstream release branch, use the
RHOAI controller, REST, and Flight image repositories and release tags in:

- `dc-controller/charts/values.yaml`
- `dc-controller/config/default/params.env`
- `config/base/params.env` (REST and Flight)

The existing kube-rbac-proxy image is not changed by controller onboarding;
release owners must confirm its approved downstream image separately.

## Infrastructure and rollout checklist

Repository changes alone do **not** provision Konflux resources or approve an
image for delivery. Coordinate the following with the RHOAI build/release team:

1. Create or confirm the Quay output repository above and the build robots'
   push/pull access.
2. Register or confirm the Application and Component on the PR and release
   clusters, with the source branch, Containerfile, context, and output image
   above. Ensure the matching build service account, Git authentication,
   `redhat-appstudio-staginguser-pull-secret`, and PQC base-image pull access
   are available in `rhoai-tenant`. Use the team's managed onboarding workflow;
   do not create ad-hoc credentials or apply copied upstream resources.
3. Merge the Containerfile fix into source `main` and `rhoai-3.7-ea.1` before
   enabling the pipelines. Merge the central pipeline changes after the build
   identities and image permissions exist. Konflux-central automation syncs the
   definitions to the component repository; confirm the new PR definition is
   present on both `main` and the release branch. If needed, use the approved
   central sync workflow rather than editing `.tekton/` manually.
4. Verify a controller PR build on p01 and a release-push build on p02, including
   all configured architectures, source image/SBOM, FIPS checks, and integration
   and enterprise-contract policy checks. Local YAML validation does not replace
   these checks.
5. Merge the release image-default changes only after the controller release
   image is available. Configure the owning RHOAI operator/bundle's controller
   related-image entry and digest nudging in its release configuration, and
   include the new component in the release snapshot and delivery configuration.
   Do not assume adding a PipelineRun or changing Helm defaults does this.
6. Validate deployment using the built controller digest and the release REST
   and Flight images. Confirm the release's approved kube-rbac-proxy image,
   pull credentials, and disconnected mirroring configuration before delivery.

Do not copy the services' `build-nudge-files` annotation unless the corresponding
controller nudge configuration has actually been added and reviewed. No such
configuration is supplied by this onboarding change.
