---
date: 2025-11-13
categories:
  - devops
  - oci
  - terraform
  - argocd
  - github
  - cicd
  - kubernetes
  - helm
  - gitops
  - automation
---
# The DevOps Odyssey, Part 6 — Closing the Loop with GitHub Auto-Tagging
##### originally posted at [LinkedIn](https://www.linkedin.com/pulse/devops-odyssey-part-6-closing-loop-github-janus-chung-lfqjc/) at November 13, 2025

In the last chapter, I left a promise — to make the system *truly GitOps-native*. To bridge the small but important gap between building images and updating manifests.

That loop is now closed.

Every time a Docker image for **Job Winner** or the **photo app** is built and pushed, GitHub Actions updates the **Argo CD repository** automatically. No manual tag edits, no pull requests waiting in the dark. The commit that produces the container now also defines its deployment.

The infrastructure finally breathes on its own.

![Throwing remote triggers to upgrade Autobot](../../assets/blog/oci-k3s-auto-tagging/banner.jpg)

<!-- more -->

## From CI to Continuous Reconciliation

The workflow starts in the **application repository** — one for Jobwinner’s backend, one for its frontend UI. Each push triggers a GitHub Action that builds and publishes a multi-architecture image to `ghcr.io`.

```yaml
# .github/workflows/deploy-jobwinner-backend.yml
on:
  push:
    paths:
      - 'job-winner/**'
      - '.github/workflows/deploy-jobwinner-backend.yml'
    branches:
      - '**'

jobs:
  docker-publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        uses: ./.github/actions/docker-build-or-tag
        with:
          token: ${{ secrets.GH_TOKEN }}
          image-name: job-winner
          dockerfile-dir: job-winner

  update-deploy-repo-tag:
    needs: docker-publish
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Update Argo CD deployment repo
        uses: ./.github/actions/update-deployment-repo
        with:
          app_name: job-winner
          deployment_type: backend
          app_id: ${{ secrets.GH_APP_ID }}
          private_key: ${{ secrets.GH_APP_PRIVATE_KEY }}
```

A similar workflow in `deploy-jobwinner-frontend.yml` does the same for the UI build. Both use the same two composite actions that form the backbone of this automation.

## The Two Actions That Make It Work

The first composite action — `docker-build-or-tag` — builds and pushes the image, tagging both `latest` and the abbreviated commit SHA.

```yaml
# .github/actions/docker-build-or-tag/action.yml
runs:
  using: composite
  steps:
    - uses: actions/checkout@v4
    - uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: januschung
        password: ${{ inputs.token }}
    - uses: docker/setup-buildx-action@v3
    - uses: docker/build-push-action@v6
      with:
        push: true
        platforms: linux/amd64,linux/arm64
        context: ${{ inputs.dockerfile-dir }}
        tags: |
          ghcr.io/januschung/${{ inputs.image-name }}:${{ env.COMMIT_SHA }}
          ghcr.io/januschung/${{ inputs.image-name }}:latest
```

Once the image is live in GHCR, the second composite action — `update-deployment-repo` — fires a `repository_dispatch` to the **Argo CD repo**, carrying a payload with the new tag, app name, and deployment type.

```yaml
# .github/actions/update-deployment-repo/action.yml
runs:
  using: composite
  steps:
    - uses: actions/create-github-app-token@v2
      with:
        app-id: ${{ inputs.app_id }}
        private-key: ${{ inputs.private_key }}
        owner: januschung
        repository: argocd

    - name: Trigger Argo CD repo dispatch
      run: |
        curl -sS -X POST \
          -H "Authorization: token ${{ steps.app-token.outputs.token }}" \
          -H "Accept: application/vnd.github.v3+json" \
          https://api.github.com/repos/januschung/argocd/dispatches \
          -d '{"event_type":"update-tag","client_payload":{"tag":"${{ env.COMMIT_SHA }}","app_name":"${{ inputs.app_name }}","deployment_type":"${{ inputs.deployment_type }}"}}'
```

## Short-Lived Trust with GitHub App Tokens

The trigger uses a **GitHub App** instead of a personal access token — and that choice matters.
Each workflow generates a short-lived installation token, valid for about an hour and scoped only to the Argo CD repository. There are no static credentials to rotate, no personal tokens hidden in secrets. It’s ephemeral trust: the token exists just long enough to update a manifest, then disappears.

It’s a small design detail, but it’s what turns automation into governance — a system that not only acts automatically, but acts responsibly.

## Updating the Argo CD Repository

In the Argo CD repository, an `update-tag.yaml` workflow listens for the `repository_dispatch` event. It checks out the repo, modifies the Helm chart `values.yaml`, creates a pull request, and auto-merges it.

```yaml
# .github/workflows/update-tag.yaml
on:
  repository_dispatch:
    types: [update-tag]

jobs:
  update-tag:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - name: Install yq
        run: |
          sudo curl -sL https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -o /usr/local/bin/yq
          sudo chmod +x /usr/local/bin/yq
      - name: Update image tag
        run: |
          cd lab/${{ github.event.client_payload.app_name }}/chart
          yq -i ".${{ github.event.client_payload.deployment_type }}.image.tag = \"${{ github.event.client_payload.tag }}\"" values.yaml
      - name: Create PR and enable auto-merge
        uses: peter-evans/create-pull-request@v6
        with:
          commit-message: "Auto update image tag to ${{ github.event.client_payload.tag }}"
          branch: auto/update-${{ github.event.client_payload.app_name }}
          delete-branch: true
```

By the time the PR merges, Argo CD detects the manifest change and redeploys the updated image automatically. The workflow becomes a heartbeat — a small, invisible cycle that keeps the system alive and current.

## Continuous Motion, Minimal Intervention

This is not about adding complexity; it’s about removing the last manual gesture.
No human now decides when “latest” becomes “running.” The repository does.

The pattern scales naturally — every app can reuse the same pair of actions, regardless of language or build context. It’s self-similar, declarative, and composable — qualities that make GitOps more than just a pipeline.

The *Jobwinner* backend, the frontend UI, and even the photo app now move in rhythm. Each push ripples across the system, quietly, predictably.

## Reflection

Each phase of this Odyssey has refined the boundaries between *tools* and *intent.*
This one erases another — the point where CI ends and deployment begins.

From here, the next chapter will look outward again — installing **Prometheus** and **Grafana**, bringing visibility and voice to this living system. Metrics, logs, and dashboards that let the cluster tell its own story.

Somewhere between a commit and a container, between Git and light, the Odyssey continues.
