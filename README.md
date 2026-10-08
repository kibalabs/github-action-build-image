# GitHub Action - Build Image

Build a docker image in one step: with a shared BuildKit when the runner has one, or with the GitHub Actions cache otherwise.

The same workflow runs fast on self-hosted runners that share a BuildKit (and its cache), and still caches its layers when it falls back to GitHub's runners. It sets up Buildx, builds, and then either only checks the image builds, makes it available to the local docker daemon, or pushes it.

## Example

```yaml
jobs:
  app-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: Build testable image
        uses: kibalabs/github-action-build-image@v1
        with:
          image: app
          context: ./app/
          target: build
          tag: app-build
      - name: Run tests
        run: docker run --rm app-build make test
      - name: Check the full image builds
        uses: kibalabs/github-action-build-image@v1
        with:
          image: app
          context: ./app/
  app-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push image
        uses: kibalabs/github-action-build-image@v1
        with:
          image: app
          context: ./app/
          push: ghcr.io/${{ github.repository }}-app:latest
```

## What it builds

Set at most one of `tag` and `push`:

- Neither: the image is only built, to check that it builds and to fill the cache.
- `tag`: the image is available to the local docker daemon under that tag, e.g. to run checks in it.
- `push`: the image is pushed to those tags. Log in to the registry first.

## Where it builds

The runner decides, through environment variables, so the same workflow works on any runner:

- `BUILDKIT_ENDPOINT` set (e.g. `tcp://buildkit:1234`): builds on that BuildKit with the `remote` driver, which keeps its own cache. Images built with `tag` are pushed uncompressed to `BUILDKIT_REGISTRY` (e.g. `buildkit:5000`), a registry the BuildKit and the runner's docker daemon can both reach, and pulled from there instead of being loaded.
- `BUILDKIT_ENDPOINT` not set (e.g. on GitHub's runners): builds in a `docker-container` builder with the GitHub Actions cache (`type=gha`, `cache-mode` defaults to `max`) scoped to `image`. Pass `driver-opts: network=host` if the build pushes to a registry on the runner's `localhost`, e.g. a service container.

Set them in the runner's environment (e.g. its `.env` file, or the runner container's environment), not in workflows, so jobs that fall back to GitHub's runners don't get them.

Every build in a job reuses the builder of the first one, so a later build starts with the layers of earlier ones.

### Choosing `image`

`image` is the cache scope, so builds that share it share their cache. A GitHub Actions cache scope only keeps what the last build wrote to it:

- Use the same `image` for builds of the same Dockerfile and build args, e.g. a check build of the `build` target and the deploy build of the full image. The full build includes the `build` stage, so what it writes covers both, and deploys on `main` fill the cache that pull requests read.
- Use a different `image` when the Dockerfile or build args change, e.g. a worker built from `worker.Dockerfile` in the same context, or the same app built with different build args. Otherwise each build replaces the other's cache.

GitHub keeps up to 10 GB of cache per repository and evicts the least recently used entries beyond that. If several large images keep evicting each other, use `cache-mode: min`, which only caches the layers of the final image.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `image` | | Name of the image these layers belong to, e.g. `api`. Used as the cache scope and as the image name in the BuildKit registry. |
| `context` | | Build context. |
| `file` | `Dockerfile` in the context | Path to the Dockerfile. |
| `target` | `''` | Build target. |
| `build-args` | `''` | Build args, one per line. |
| `tag` | `''` | Make the built image available to the local docker daemon under this tag. |
| `push` | `''` | Tags to push the built image to, one per line. |
| `cache-mode` | `max` | GitHub Actions cache mode without a shared BuildKit: `max` caches every stage, `min` only the final image's layers. |
| `driver-opts` | `''` | Options for the `docker-container` builder used without a shared BuildKit, one per line, e.g. `network=host`. |
| `summary` | `false` | Add the build summary to the job summary and upload the build record as an artifact. |

## Outputs

| Output | Description |
| --- | --- |
| `digest` | Digest of the built image, when it is tagged or pushed. |

## With Skip If Passed

When [Skip If Passed](https://github.com/kibalabs/github-action-skip-if-passed) decides a job can be skipped, it sets `CHECKS_ALREADY_PASSED=true` and this action does nothing.

## Development

The action is the composite action in `action.yml`. The Build workflow runs it on pull requests twice: once with the GitHub Actions cache and once with a BuildKit and registry started in the job.

To release, bump `version` in `package.json` in a PR, then run the Release workflow on `main` (Actions → Release → Run workflow). It tags `vX.Y.Z`, publishes the release and moves the `vX` tag to it, so `@v1` always points at the latest 1.x release. Versions with a pre-release suffix (e.g. `1.1.0-rc1`) are published as pre-releases and don't move `vX`. Only the release workflow can push version tags.
