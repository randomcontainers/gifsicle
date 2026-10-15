# gifsicle

Container images with [Gifsicle](https://www.lcdf.org/gifsicle/), which optimizes and edits GIF images and animations. `gifsicle` makes GIF files smaller, losslessly or with lossy compression, reduces their colors, resizes, crops and rotates them, and adds, removes or reorders frames. `gifdiff` checks whether two GIFs look the same. Both are compiled from the release tag on Ubuntu and Alpine. The default image also contains FFmpeg, for turning video clips into GIFs that gifsicle then shrinks. The images are rebuilt when Gifsicle tags a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the Gifsicle project. Report problems with the image in this repository and problems with Gifsicle itself [upstream](https://github.com/kohler/gifsicle/issues).

## Quick start

Optimize an animation in the current directory:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/gifsicle -O3 input.gif -o output.gif
```

`-O3` stores only the part of each frame that changes, trying several methods, and the animation looks exactly as before. `-o` names the output file; `--batch` (`-b`) writes each result back over its input instead:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/gifsicle --batch -O3 *.gif
```

`--lossy` lets the compressor change some pixels to make the file smaller, and `--colors` reduces the color table. Higher `--lossy` values give smaller files with more noise; without a value it is 20:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/gifsicle -O3 --lossy=80 --colors 64 input.gif -o small.gif
```

Resize to a width of 320 pixels, keeping the aspect ratio, or shrink to fit a box:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/gifsicle --resize-width 320 input.gif -o resized.gif
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/gifsicle --resize-fit 400x400 input.gif -o thumb.gif
```

`--info` prints the frames of a GIF with their sizes, positions and delays, and `--color-info` adds the color tables:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/gifsicle --info input.gif
```

The entrypoint runs `gifsicle` under `tini`. For `gifdiff`, override it. gifdiff prints nothing and exits with 0 when the two GIFs look the same, and lists the differences and exits with 1 when they do not:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint gifdiff \
  ghcr.io/randomcontainers/gifsicle input.gif output.gif
```

The [Gifsicle manual](https://www.lcdf.org/gifsicle/man.html) describes every option, and `gifsicle --help` lists them.

## Frames

A frame selection after a file name picks frames from it, counting from 0: `'#0'`, `'#2-5'`, `'#3-'` or `'#-1'` for the last one. Quote them, since the shell reads `#` as the start of a comment:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/gifsicle input.gif '#0-9' -o first-ten.gif
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/gifsicle input.gif --delete '#0' -o rest.gif
```

`--explode` writes each frame to a file of its own, named after the input or `-o` with the frame number appended (`frame.000`, `frame.001`, ...). Several GIFs given together are merged into one animation; `--delay` sets the time per frame in hundredths of a second and `--loopcount` makes it repeat:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/gifsicle --explode input.gif -o frame
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/gifsicle --delay 10 --loopcount=forever frame.* -o animation.gif
```

## GIFs from video

The default image has FFmpeg, which makes the GIF from a video clip. Its `palettegen` filter picks a color table for the whole clip and `paletteuse` applies it, which gives better colors than FFmpeg's fixed default palette. `fps` and `scale` keep the frame rate and size down. gifsicle then shrinks the result:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh \
  ghcr.io/randomcontainers/gifsicle -c \
  'ffmpeg -loglevel error -i clip.mp4 -vf "fps=10,scale=480:-1:flags=lanczos,split[a][b];[a]palettegen[p];[b][p]paletteuse" clip.gif && gifsicle -O3 --lossy=80 clip.gif -o clip-small.gif'
```

`-ss <seconds>` and `-t <seconds>` before `-i` cut a part of the clip. This needs the default image, since `slim` has no FFmpeg.

## What is in the image

| Area | Details |
|---|---|
| Programs | `gifsicle` and `gifdiff` |
| Features | Every option of `gifsicle`, including `--lossy`, color reduction with `--colors` and `--use-colormap`, and `--threads`, which resizes the frames of an unoptimized GIF in parallel |
| Default image | FFmpeg (`ffmpeg` and `ffprobe`), to make GIFs from video and read GIFs back into other formats |

Not included:

- `gifview`, the X11 viewer: the images have no display.
- The man pages.

Gifsicle needs only the C library. The configure options, the compiler and the commit are in `/usr/local/share/randomcontainers/gifsicle/buildinfo`.

## Default or slim

Use the default image (`latest`) to make GIFs from video: it adds FFmpeg (see [GIFs from video](#gifs-from-video)). `slim` has only Gifsicle. Use it to optimize or edit GIF files you already have, or as the base for your own image.

The default image includes FFmpeg, which is licensed under the GNU General Public License (GPL-3.0-or-later). The default image is also published as `ghcr.io/randomcontainers/gifsicle-ffmpeg`, built in the [gifsicle-ffmpeg](https://github.com/randomcontainers/gifsicle-ffmpeg) repository with the same contents and a different digest.

## Tags

`<version>` is a Gifsicle release such as `1.98`, and `<major>` is its first number, `1`, which follows the newest release. Most Gifsicle releases have two numbers. A point release such as `1.98.1` would also take the shorter `1.98` tags: `1.98`, `1.98-slim`, `1.98-alpine` and the other `1.98` tags without a distro version would move to it, while the `1.98` tags ending in `ubuntu26.04` or `alpine3.24` stay on 1.98.

| Default (with FFmpeg) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<major>` | `slim`, `<version>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Gifsicle version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Without file names, gifsicle reads a GIF from standard input and writes the result to standard output, so no mount is needed. It refuses to read or write GIF data on a terminal, so pass `-i` without `-t`:

```sh
docker run --rm -i ghcr.io/randomcontainers/gifsicle -O3 < input.gif > output.gif
```

## Untrusted files

Gifsicle reads GIF files with its own parser, written in C, and fuzzing has found memory-safety bugs in it. For GIFs from unknown sources, mount the input read-only, write to a separate directory and take away what the container does not need:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD/uploads:/in:ro" -v "$PWD/out:/work" \
  --network none --read-only --cap-drop ALL --security-opt no-new-privileges \
  --memory 512m --pids-limit 16 \
  ghcr.io/randomcontainers/gifsicle -O3 /in/upload.gif -o output.gif
```

`--transform-colormap` runs a shell command, so do not pass options to gifsicle from untrusted input either.

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no FFmpeg, so FFmpeg updates do not rebuild it. The packages Gifsicle needs are listed in `/usr/local/share/randomcontainers/gifsicle/runtime-deps`. This image adds a script that optimizes the GIFs it is given in place:

```dockerfile
FROM ghcr.io/randomcontainers/gifsicle:slim-alpine@sha256:...
COPY --chmod=755 shrink.sh /usr/local/bin/shrink
ENTRYPOINT ["tini", "--", "shrink"]
```

```sh
#!/bin/sh
# shrink.sh: optimize each GIF given, in place, with light lossy compression
set -eu
exec gifsicle --batch -O3 --lossy=30 "$@"
```

The image runs as UID 1000, so switch to `USER root` to install distro packages, then back to `USER 1000:1000`. The entrypoint of the slim image is `["tini", "--", "gifsicle"]`. To pick up new Gifsicle releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/gifsicle:latest \
  --repo randomcontainers/gifsicle --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/gifsicle-ffmpeg` are built in that repository, so verify them with `--repo randomcontainers/gifsicle-ffmpeg` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/gifsicle:latest --format '{{ json .SBOM }}'
```

Gifsicle is compiled here, so it is not in the SBOM and image scanners do not check it. Its version and commit are in `/usr/local/share/randomcontainers/gifsicle/buildinfo`.

Gifsicle's release tarballs on lcdf.org have no checksums or signatures, the project publishes no GitHub releases, its tags are lightweight and its commits are not signed, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [kohler/gifsicle](https://github.com/kohler/gifsicle) and stops unless the tag points to the commit recorded in `package.yml`, so a tag that is moved after the version was picked up fails the build. It also runs Gifsicle's test suite on each distro and platform.

## Updates

The project checks the `v<version>` tags of [kohler/gifsicle](https://github.com/kohler/gifsicle/tags) every 15 minutes. A release is picked up once its tag is 24 hours old. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built. Beta tags such as `v1.15b` are skipped.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new FFmpeg image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t gifsicle:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

Gifsicle, gifdiff and the LCDF GIF library they are built on are licensed under the GNU General Public License, version 2 (GPL-2.0-only). Gifsicle's own README adds an alternative license, and CLP, the command line parser compiled into both programs, may also be used under the BSD-like [Click license](https://github.com/kohler/click/blob/master/LICENSE). The image uses all of it under the GPL-2.0, so the license label is `GPL-2.0-only`.

`/usr/local/share/randomcontainers/gifsicle/licenses/` has Gifsicle's `COPYING` with the GPL-2.0 text and CLP's notice from `src/clp.c` as `NOTICE.clp`. The Ubuntu and Alpine packages in the image, such as the C library, tini and ca-certificates, keep their own licenses. The SBOM lists them.

The corresponding source for each image:

- Gifsicle: every version has a GitHub release in this repository, named `v<version>`, with `gifsicle-<version>.tar.gz`, a `git archive` of the commit that was compiled. The build applies no patches; it generates the configure script from `configure.ac` and the `Makefile.am` files in that archive with `autoreconf`. `/usr/local/share/randomcontainers/gifsicle/source` lists that release and the commit on GitHub.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure option.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The default image adds FFmpeg, licensed under GPL-3.0-or-later; its license files and corresponding source are described in the [ffmpeg](https://github.com/randomcontainers/ffmpeg#licenses) repository.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
