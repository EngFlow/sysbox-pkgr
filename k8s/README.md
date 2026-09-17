Sysbox-deploy-k8s image generation and update procedure
=======================================================

1) Build the sysbox-deploy images through the usual `make <target>` method.

  NOTE: The process must be completed in every supported architecture (i.e.,
  amd64 and arm64).

  NOTE: this build does *not* cross-compile. It downloads and runs an
  arch-specific Go toolchain and C compiler inside the build containers, so
  those containers must actually execute as the target architecture, not
  just produce binaries for it. If the build host's native architecture
  doesn't match the one you're building for, install QEMU emulation support
  once per host:

```
$ docker run --rm --privileged tonistiigi/binfmt --install arm64
```

  Then force the target architecture for the whole build -- the Makefile
  otherwise auto-detects `SYS_ARCH` from `uname -m`, which is wrong when
  cross-building:

```
$ export DOCKER_DEFAULT_PLATFORM=linux/arm64
$ make SYS_ARCH=arm64 sysbox-deploy-k8s-image
```

  NOTE: the CRI-O build step (`docker run --rm -v $(pwd)/bin:/mnt/results
  crio-bld`) writes its output as `root`, since it runs inside a container.
  If the next step (`fetch-sysbox-ce-bins`) fails with `Permission denied`
  creating `bin/sysbox-ce`, fix ownership and re-run -- the already-built
  CRI-O binaries are cached under `bin/crio`, so this only repeats the
  remaining steps, not the CRI-O build itself:

```
$ sudo chown -R $(whoami) bin/
$ make SYS_ARCH=arm64 sysbox-deploy-k8s-image
```

  NOTE: pulling the `ghcr.io/nestybox/centos7/systemd` base image requires
  read access to nestybox's private packages. If this fails with `denied:
  denied` (even on a plain `docker pull` of that image, outside the build),
  the build host's Docker credentials/policy need to allow it -- this is
  unrelated to whether the target `sysbox-deploy-k8s` image itself has been
  published, and is easy to mistake for that.

2) Before building the *next* architecture, move the current `bin/`
directory aside (e.g. `mv bin bin-amd64`). The Makefile reuses `bin/crio` if
present, so an unmoved `bin/` from one architecture's build will silently be
reused -- and be wrong -- for the next one.

3) Identify the image that has been created and re-tag it accordingly to
match the platform architecture being used, before building the next
architecture (the Makefile always produces the same, unsuffixed tag, so an
untagged second build silently overwrites the first one's tag locally --
the underlying image is still there by digest, but you'd have to dig it out
again).

  NOTE: this must be done for each supported platform.

```
$ docker images
REPOSITORY                           TAG        IMAGE ID       CREATED              SIZE
ghcr.io/nestybox/sysbox-deploy-k8s   v0.7.1-0   ebab4402b232   About a minute ago   2.17GB

$ docker tag ghcr.io/nestybox/sysbox-deploy-k8s:v0.7.1-0 ghcr.io/nestybox/sysbox-deploy-k8s:v0.7.1-0_amd64
```

  Repeat on the arm64 build to produce the `_arm64`-suffixed tag.

4) Push each image to ghcr.io (for both supported platforms):

```
$ docker push ghcr.io/nestybox/sysbox-deploy-k8s:v0.7.1-0_amd64
$ docker push ghcr.io/nestybox/sysbox-deploy-k8s:v0.7.1-0_arm64
```

5) Now is time to combine the per-architecture images into a single
multi-arch manifest. This step can be completed on any linux machine with
registry access -- it doesn't need to be one of the machines used to build
the images, and it works purely against what's already been pushed.

  Use `docker buildx imagetools create` rather than the older `docker
  manifest create`/`docker manifest push`. Modern `docker build` (BuildKit)
  attaches a build-provenance attestation to each image, wrapping it in an
  OCI index rather than a flat manifest, and `docker manifest create`
  doesn't handle that nesting correctly. `imagetools create` does, and
  produces a clean combined index pointing at the right per-arch manifests
  (the attestations ride along as extra `platform: unknown/unknown`
  entries, which is expected and harmless -- image-pulling clients select
  the manifest matching their platform and ignore the rest).

```
$ docker buildx imagetools create \
    -t ghcr.io/nestybox/sysbox-deploy-k8s:v0.7.1-0 \
    ghcr.io/nestybox/sysbox-deploy-k8s:v0.7.1-0_amd64 \
    ghcr.io/nestybox/sysbox-deploy-k8s:v0.7.1-0_arm64

$ docker buildx imagetools create \
    -t ghcr.io/nestybox/sysbox-deploy-k8s:latest \
    ghcr.io/nestybox/sysbox-deploy-k8s:v0.7.1-0_amd64 \
    ghcr.io/nestybox/sysbox-deploy-k8s:v0.7.1-0_arm64
```

6) Verify in the Github web-portal that the image digests of both the
manifest and the images fully match those just pushed:

```
$ docker buildx imagetools inspect ghcr.io/nestybox/sysbox-deploy-k8s:v0.7.1-0
$ docker buildx imagetools inspect ghcr.io/nestybox/sysbox-deploy-k8s:latest
```

  NOTE: `registry.nestybox.com/nestybox/sysbox-deploy-k8s` mirrors
  `ghcr.io/nestybox/sysbox-deploy-k8s` (confirmed identical digests) -- once
  pushed to ghcr.io, no separate publish step is needed for
  `registry.nestybox.com`.
