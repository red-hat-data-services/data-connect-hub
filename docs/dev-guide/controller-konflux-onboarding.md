# Downstream controller Konflux onboarding

The RHOAI controller build is separate from the upstream ODH controller build.
Inherited PipelineRuns targeting `open-data-hub-tenant`, the upstream repository,
or `quay.io/opendatahub/` do not onboard a downstream release.

## Process entry point

Follow the **RHOAI Component Image Onboarding Guide** (Jan 2025, In-Progress).
The supported entry point is `/create-component-onboarding-jira` from the
`aiops-skills` plugin in the skills-registry marketplace. Reuse an existing
onboarding Jira URL if one already exists; otherwise the skill clones the
RHOAI template `RHOAIENG-17225`.

The skill collects and confirms the inputs, validates
`component_onboarding_details.yaml` against its schema, attaches it to Jira,
and applies the `component-onboarding` label. That Jira attachment is the
single source of truth for the onboarding automation, not these pipeline PRs.
The scheduled GitLab CI job runs every two hours, invokes
`onboard-konflux-components-for-odh-and-rhoai`, raises the required PRs/MRs,
and records progress on Jira. DevOps review is mandatory; the automation does
not wait for reviews or merge unreviewed changes.

Do not replace this flow with the guide's deprecated manual provisioning
instructions. Share existing source/pipeline PRs on the Jira so DevOps can
reuse them rather than creating duplicate work. An Atlassian Jira component,
if needed, is created separately; it is not a Konflux Component resource.

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
| Dockerfile | `dc-controller/Dockerfile.konflux` |
| Build context | `.` (repository root) |
| Dependency prefetch | `[{"type": "gomod", "path": "dc-controller"}]` |

PR builds run on `stone-prod-p01` and publish `on-pr-{{pull_request_number}}`
images for x86_64 and arm64, expiring after 30 days. Release pushes run on
`stone-prod-p02` and publish `rhoai-3.7-ea.1` and branch/commit tags for x86_64,
arm64, ppc64le, and s390x. Both builds are hermetic and produce source images.
The controller uses the pipeline's blocking FIPS-check default; the Rust
services' FIPS exception must not be copied to it.

The base component name is `odh-data-connect-hub-controller`; the release
suffix belongs to the Konflux Component identity. Both the build and delivery
repository basename are `odh-data-connect-hub-controller-rhel9`. Delivery
targets are `registry.redhat.io/rhoai/odh-data-connect-hub-controller-rhel9`
and its `registry.stage.redhat.io` counterpart.

The Jira input still needs the release owner's confirmed release category
(GA, Tech Preview, or Beta), parent feature, approved descriptions, and
operator-manifest source/destination paths. Do not infer the release category
from the EA branch name or publish an attachment with placeholders.

## Source prerequisites

- The downstream fork exists and its default branch is `main`; REST and Flight
  already use it, so no additional fork is needed for the controller.
- The controller Dockerfile basename starts with `Dockerfile.konflux`.
- Both default base/builder images are UBI9-based Red Hat registry images pinned
  by SHA digest. The Go toolset pin is the multi-architecture manifest-list
  digest, not an architecture-specific child manifest.
- The final image defines `name`, `com.redhat.component`, `summary`,
  `description`, `maintainer`, `io.k8s.display-name`, and `io.k8s.description`.
  Its `name` label is `rhoai/odh-data-connect-hub-controller-rhel9`.
- Go dependencies are covered by `dc-controller/go.mod` and `go.sum` and the
  `gomod` prefetch above. This Dockerfile installs no additional RPM or Python
  packages, so it does not need the Rust services' RPM prefetch configuration.
- Confirm the platform team's operator installation/manifest integration.
- Existing repository-level Renovate and source-sync onboarding can be reused
  after verification; adding a second image does not require a second repo.

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

## Automation/review and delivery gates

Repository changes alone do **not** provision Konflux resources or approve an
image for delivery. Track these outcomes on the onboarding Jira; the guide's
legacy sections describe underlying resources, not a parallel manual process:

1. Quay build-repository configuration and permissions in `app-interface`,
   delivery-repository configuration in `pyxis-repo-configs`, and RHOAI product
   listing membership with the approved release category/content stream.
2. Konflux Component/Application/image-repository resources, build service
   account, Git/pull secrets, PQC base-image access, and Konflux GitHub App
   integration. Review the automation's `konflux-release-data` MR and stage/prod
   ReleasePlanAdmission mappings for this release.
3. Source prerequisites merged and central pipeline changes reviewed/synced.
   The new Dockerfile must exist on the source target branches before the
   central validator can pass its remote Dockerfile-path check. Confirm the PR
   definition reaches both source `main` and the release branch. Do not edit
   component-repository `.tekton/` files directly.
4. A green controller build on every configured release architecture, including
   source image/SBOM, FIPS, integration, and enterprise-contract policy checks.
   Local schema validation is not evidence of a green component image build.
5. Only after a real green image exists, use its actual digest for
   `RELATED_IMAGE_ODH_DATA_CONNECT_HUB_CONTROLLER_IMAGE` in the release's
   `RHOAI-Build-Config` bundle patch. Review build-config repository mappings,
   image-mirror configuration where still required, release snapshot/delivery,
   and controller-to-operator digest nudging. Never invent a seed digest.
6. Operator integration in `rhods-operator/manifests-config.yaml`, synced from
   main to the release branch, including the controller's RHOAI image mapping.
   Its existing ODH-only related-image entry is not sufficient. Leave
   `operands-map.yaml` to the build automation. Review tracer component labels
   for the bundle and supported-OCP catalog Dockerfiles.
7. After the preceding changes are reviewed/merged, an explicitly authorized
   operator-processor run and green operator, bundle, and FBC builds, followed
   by scrum-team validation of the resulting FBC. Do not deploy the proposed
   Helm/Kustomize release defaults before the controller image is available.
8. Future-release GitOps component/image-repository onboarding templates. Do
   **not** add the newly onboarded component to the current release's GitOps
   template outside the prescribed onboarding flow; the guide warns this can
   make GitOps out of sync.

Confirm the release's approved kube-rbac-proxy image, pull credentials, and
disconnected mirroring configuration before delivery. Existing source/pipeline
PRs are only part of this checklist and must not be described as completed
onboarding.

Do not copy the services' `build-nudge-files` annotation unless the corresponding
controller nudge configuration has actually been added and reviewed. No such
configuration is supplied by this onboarding change.
