# Build OCI GitHub Actions Workflow

> [!IMPORTANT]
>
> This workflow is deprecated. It depends on the devtools images in
> [engineering-docker-images](https://github.com/coopnorge/engineering-docker-images)
> (one docker compose service per repo), which are being deleted. The workflow
> will be removed together with them. Progress is tracked in
> <https://github.com/coopnorge/engineering-docker-images/issues/4456>.
>
> Go repositories can move now: use
> [`goapp.yaml`](https://github.com/coopnorge/mage/blob/main/.github/workflows/goapp.yaml)
> from `coopnorge/mage`, which builds and pushes the app image. Libraries use
> [`mage.yaml`](https://github.com/coopnorge/mage/blob/main/.github/workflows/mage.yaml).
> There is no generic mage replacement for other stacks yet, so those
> repositories have no migration path today. That work is tracked in
> <https://github.com/coopnorge/mage/issues/923>.

Usage docs:
<https://inventory.internal.coop/docs/default/component/github-workflow-devtools-build-oci>
or [./docs](./docs).
