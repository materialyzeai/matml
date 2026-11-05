# Building and pushing images

Images are named after their Dockerfile: `docker/Dockerfile.<name>` is tagged `materialyzeai/<name>:latest`,
and `docker/Dockerfile.<name>.<variant>` is tagged `materialyzeai/<name>:latest-<variant>`.

CI builds and pushes each image when its Dockerfile changes on `main` (see `.github/workflows/`). The
kokkos-cpu-based images (`matgl:latest-kokkos-cpu`, `matml:latest`, `matml:latest-kokkos-cpu`) use the
reusable workflow `build-multiarch.yml`: each architecture is built and smoke-tested on a native amd64 or
arm64 runner, pushed by digest, and the digests are merged into one multi-arch tag. These workflows also
build and smoke-test on pull requests, without pushing. The other images are built for both architectures
under QEMU. `matgl:latest-kokkos` (CUDA) and the legacy `-dgl` images have no workflow; build them locally if needed.

To build an image locally,

```shell
docker build -t materialyzeai/matml:latest -f docker/Dockerfile.matml .
```

Pushing the image to the repository.

```shell
docker push materialyzeai/matml:latest
```

The `invoke` tasks in `tasks.py` do both:

```shell
uv run invoke publish docker/Dockerfile.matml   # one image
uv run invoke publish-all                       # every docker/Dockerfile.*
```

Derived images build `FROM` the published base images (e.g. `matml:latest` and
`matml:latest-kokkos-cpu` from `matgl:latest-kokkos-cpu`), so publish the base image first.
