# Build OCI Image GitHub Actions Workflow

The workflow will validate, build and optionally public an OCI Image. The
workflow requires a devtools compliant OCI Image with the executables
`validate`, `build` and `publish`.

## Usage

The example uses our Go Lang devtools.

### Inputs

| Name                         | Type    | Required | Default | Description |
|------------------------------|---------|----------|---------|-------------|
| `publish`                    | boolean | optional | `false` | Set to true to publish the OCI Image produced by this workflow. |
| `docker-compose-service`     | string  | required | N/A     | The docker compose service providing the CI executables validate, build and publish.                     |
| `workload_identity_provider` | string  | required | N/A     | |
| `service_account`            | string  | required | N/A     | |

### Outputs

| Name        | Type   | Description                            |
|-------------|--------|----------------------------------------|
| `image`     | string | The full OCI Image reference with tag. |
| `image_tag` | string | The OCI Image tag.                     |

### Workflow configuration

```yaml title=".github/workflows/cicd.yaml"
---
name: CI/CD
on:
  push

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  golang-ci:
    permissions:
      contents: read
      id-token: write
      packages: read
    name: Go
    uses: coopnorge/github-workflow-devtools-build-oci/.github/workflows/oci-ci.yaml@v1
    with:
      docker-compose-service: golang-devtools
      publish: ${{ github.ref == 'refs/heads/main' }}
      workload_identity_provider: projects/889992792607/locations/global/workloadIdentityPools/github-actions/providers/github-actions-provider
      service_account: helloworld-github-actions@helloworld-shared-0918.iam.gserviceaccount.com
    secrets: inherit

  update-infrastructure:
    needs:
      - golang-ci
    if: ${{ github.ref == 'refs/heads/main' }}
    uses: coopnorge/engineering-github-actions/.github/workflows/update-infrastructure-repo.yaml@main
    secrets:
      write-back-app-pem: ${{ secrets.WRITEBACK_APP_PRIVATE_KEY_PEM }}
      approve-pr-token: ${{ secrets.REVIEWBOT_GITHUB_TOKEN }}
    with:
      environment-matrix-json: >-
        [{
            "environment": "staging",
            "auto-merge": "true" ,
            "auto-approve" : "true"
          },
          {
            "environment": "production",
            "auto-merge": "true" ,
            "auto-approve" : "false"
        }]
      source-github-repo: coopnorge/helloworld-infrastructure
      update-script: |
        set -x
        wget https://github.com/mikefarah/yq/releases/download/v4.20.1/yq_linux_amd64 -O ${GITHUB_WORKSPACE}/yq
        chmod +x ${GITHUB_WORKSPACE}/yq

        wget -O- https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv4.5.2/kustomize_v4.5.2_linux_amd64.tar.gz | tar -C ${GITHUB_WORKSPACE} -zxvf -
        chmod +x ${GITHUB_WORKSPACE}/kustomize

        pushd kubernetes/kustomize-deployments/${environment}/
        ${GITHUB_WORKSPACE}/yq -i e '.spec.template.metadata.labels."tags.datadoghq.com/version" = "${{ needs.golang-ci.outputs.image_tag }}"' deployment.yaml
        ${GITHUB_WORKSPACE}/yq -i e '.spec.template.metadata.labels.version = "${{ needs.golang-ci.outputs.image_tag }}"' deployment.yaml
        ${GITHUB_WORKSPACE}/yq -i e '.metadata.labels."tags.datadoghq.com/version" = "${{ needs.golang-ci.outputs.image_tag }}"' deployment.yaml
        ${GITHUB_WORKSPACE}/yq -i e '.metadata.labels.version = "${{ needs.golang-ci.outputs.image_tag }}"' deployment.yaml
        ${GITHUB_WORKSPACE}/kustomize edit set image europe-docker.pkg.dev/helloworld-shared-0918/helloworld/helloworld="${{ needs.golang-ci.outputs.image }}"
        popd

  build:
    needs:
      - golang-ci
      - update-infrastructure
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: exit 1
        name: "Catch errors"
        if: |
          needs.golang-ci.result == 'failure' ||
          needs.update-infrastructure.result == 'failure'
```

### Docker Compose Configuration

```yaml title="docker-compose.yaml"
version: "3.7"

services:
  golang-devtools:
    build:
      context: docker-compose
      target: golang-devtools
      dockerfile: Dockerfile
    privileged: true
    security_opt:
      - seccomp:unconfined
      - apparmor:unconfined
    volumes:
      - .:/srv/workspace:z
      - ${DOCKER_CONFIG:-~/.docker}:/root/.docker
      - ${GIT_CONFIG:-~/.gitconfig}:${GIT_CONFIG_GUEST:-/root/.gitconfig}
      - ${SSH_CONFIG:-~/.ssh}:/root/.ssh
      - ${XDG_CACHE_HOME:-xdg-cache-home}:/root/.cache
    environment:
      SERVICE_PORT: ":50051"
      SHUTDOWN_TIMEOUT: 5s
      GOMODCACHE: /root/.cache/go-mod
    ports:
      - "50051:50051"
    networks:
      - default
    working_dir: /srv/workspace
    command: "golang-run"

networks:
  default:
volumes:
    xdg-cache-home: {}
```

### Docker compose Dockerfile

```Dockerfile title="docker-compose/Dockerfile"
FROM ghcr.io/coopnorge/engineering-docker-images/e0/devtools-golang-v1beta1:latest@sha256:07d1f3058917c93ed51a2d8bc3bfefe6008e5132282b259327462127d781f2ba AS golang-devtools
```

### devtools Environment Configuration

```shell title="devtools.env"
APP_DOCKERFILE=/usr/local/share/devtools-golang/Dockerfile.app
APP_NAME=helloworld
GOFLAGS=-mod=readonly
OCI_REF_NAMES=europe-docker.pkg.dev/helloworld-shared-0918/helloworld/helloworld
```
