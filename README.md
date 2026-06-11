# `pymage`

<img width="300" height="300" alt="Screenshot 2026-06-10 at 9 49 00 AM" src="https://github.com/user-attachments/assets/9112e76a-276e-4530-9324-a0dc5b0cd726" />

_[Pym discs](https://marvel.fandom.com/wiki/Pym_Discs) make things smaller._

[![pymage](https://github.com/imjasonh/pymage/actions/workflows/ci.yaml/badge.svg)](https://github.com/imjasonh/pymage/actions/workflows/ci.yaml)

`pymage` is a dockerless, unprivileged, layer-aware and fast container image builder for Python applications using `uv`.

It's built in the spirit of [`ko`](https://ko.build) for Go, [`jib`](https://github.com/googlecontainertools/jib) for Java, [`krust`](https://github.com/imjasonh/krust) for Rust and others.

Like `jib` and unlike `ko` and `krust`, `pymage` splits Python dependencies into separate layers, separate from the application code, so changing your code without changing a dependency only updates one layer, and updating a dependency only updates one layer. If your base image changes, only those layers are updated. And if nothing changes, there's nothing to do!

This has benefits at both push-time and pull-time -- when a Kubernetes node already running `:v1.2.3` pulls `:v1.2.4`, only the new layers will need to be fetched. This can have significant benefit for deployment times, and registry storage and egress costs.

`pymage` doesn't run containers to perform the build, it just writes application source code to the registry, and copies dependencies' existing compiled wheels to the registry. If any dependency doesn't have an available wheel for the specified platform(s), the build fails. `pymage` will never run any code at build-time.

Because `pymage` is just moving wheels around, it supports multi-platform builds by just moving those platforms' wheels around. `pymage` runs on Linux, macOS and Windows.

See [`DESIGN.md`](./DESIGN.md) for the full rationale.

This design pays off most for **large AI/GPU images**, where for example `torch` wheels carry the CUDA runtime (`nvidia-*` wheels) and the dependencies are ~98% of the image.
Bumping one dependency re-uploads a single ~11 MB layer instead of re-pushing the whole ~2.9 GB `uv sync` venv layer.

See [`docs/ai-image-comparison.md`](./docs/ai-image-comparison.md). For pure-wheel
web apps see [`docs/real-world-comparison.md`](./docs/real-world-comparison.md).

## Installation

```
go install github.com/imjasonh/pymage@latest
```

(TODO: real releases, if anybody wants them)

## Usage

pymage is designed for projects that use [`uv`](https://docs.astral.sh/uv/). Configure it
once in `pyproject.toml`:

```toml
[tool.pymage]
repo = "registry.example.com/me/myapp"
```

Then, from the project root:

```
pymage build              # builds + pushes registry.example.com/me/myapp:latest
pymage build -t v1.2.3    # ...:v1.2.3 (and -t is repeatable for multiple tags)
pymage build ./example    # build a different project directory (positional arg)
```

The project directory is the first positional argument (default: the current directory).

`pymage` prints the resulting `repo@sha256:...` reference to stdout, so you can run the image directly:

```
docker run "$(pymage build)"
```

`-t/--tag` also tags the image. If no tag is specified, the image is not tagged and is only pushed by digest.

### Configuration

`[tool.pymage]` keys mirror the build flags; an explicit flag always overrides
the config value, which overrides the built-in default.

| Key | Flag | Default |
| --- | --- | --- |
| `repo` | `--repo` | `$PYMAGE_REPO`, else *(required to push)* |
| `tags` | `-t`/`--tag` (repeatable) | none |
| `base` | `--base` | `cgr.dev/chainguard/python:latest` |
| `platforms` | `--platform` | the platforms the **base image** supports (default: `linux/amd64` and `linux/arm64`) |
| `layer-strategy` | `--layer-strategy` | `auto` |
| `max-layers` | `--max-layers` | `127` |
| `max-wheel-layers` | `--max-wheel-layers` | *(derived from `max-layers`)* |
| `push-concurrency` | `--push-concurrency` | auto (scales with CPUs) |
| `no-cache` | `--no-cache` | `false` (caching is on by default) |
| `extras` | `--extra` (repeatable) | -- (enables `uv` project optional-dependency groups) |
| `package` | `--package` | -- (build a single `uv` workspace member) |
| `python` | `--python` | auto-detected from the base image |
| `prefix` | `--prefix` | `/app/.venv` |
| `workdir` | `--workdir` | `/app` |
| `user` | `--user` | *(base default)* |
| `entrypoint` | `--entrypoint` | `[project.scripts]` console script |
| `cmd` | `--cmd` | -- |
| `env` | `--env` | `PYTHONPATH=/app/src` when `src/` exists |
| `labels` | `--label` | *(base default)* |
| `find-links` | `--find-links` | download wheels from the lock |

Other defaults: the source directory is the first positional argument (default
`.`); the lock is `uv.lock` in that directory (falling back to
`requirements.txt`). Wheels are fetched over the network from the lock URLs on
first use and cached by SHA-256 in `~/.cache/pymage/wheels`; set `find-links` to
a local wheel directory for offline / air-gapped builds.

See [`example/`](./example/) for a FastAPI app with a `[tool.pymage]` table you
can build as-is:

```
go run . build ./example --repo ttl.sh/pymage -t latest
```

### Layering

By default (`layer-strategy = "auto"`) `pymage` keeps **one layer per wheel** for
maximum incrementality, as long as the total image stays within a layer budget -- `127`
layers by default (`max-layers`, counting the base image's layers, the
dependency layers, and the app source layer). Set `max-wheel-layers` to cap the
dependency layers directly.

Some container runtimes enforce a cap on the total number of layers. The default config for `pymage` results in as many layers as possible, staying under that cap.

When there are more wheels than the budget allows, `pymage` bin-packs them by
hashing each distribution's (normalized) name to a stable bucket. Because a
wheel's bucket depends only on its name, adding, removing, or version-bumping a
single dependency only changes _that one bucket's layer_ -- every other layer
keeps its digest and is reused. (`per-wheel` forces one layer per wheel with no
cap; `single-deps-layer` puts everything in one layer.)

### Optional dependencies, workspaces, and markers (uv.lock)

pymage installs the project's runtime closure from `uv.lock` (the deps you'd
get from `uv sync --no-dev`), not every package in the lock. When `uv` is
installed and a `pyproject.toml` is present, resolution is delegated to
`uv export --frozen --no-dev` -- `uv` is the source of truth for the closure,
extras, groups, and workspace selection.

`pymage` then evaluates the environment markers `uv` emits for the target
and attaches wheel URLs from the lock. `uv` is therefore required to build
from a `uv.lock` -- if it isn't installed the build fails. As an alternative, export a
hashed `requirements.txt` and build with `--lock requirements.txt --find-links`.
The selectors below map onto `uv export` flags:

- `--extra <group>` enables one of the project's own
  `[project.optional-dependencies]` groups (repeatable). Extras requested by
  your dependencies (e.g. `fastapi[standard]`) are always followed.
- `--package <name>` roots the closure at a single `uv` workspace member
  instead of the union of all members -- useful for monorepos that build several
  images from one lock.
- Environment markers are evaluated for the target. A dependency gated on
  `sys_platform == 'win32'` or `python_version < '3.11'` is included only when it
  applies to the image platform or interpreter being built, so Linux images don't carry
  Windows-only or stale-Python-only packages. Markers are evaluated per platform,
  so each arch of a multi-arch build gets the correct set.

### Source distributions (sdists)

pymage installs _pre-built wheels only_ -- it does not attempt to build sdists. This is a
deliberate choice: building a source distribution runs the dependency's own build
code (`setup.py` / build backend) on the build host, which would break `pymage`'s
core guarantees -- it's not hermetic or byte-reproducible, it's a remote-code-
execution surface with no container to sandbox it, it needs a build toolchain,
and compiled packages can only be built for the host architecture (breaking multi-arch builds).

In practice this is rarely an issue: the modern Python ecosystem is wheel-first, so
mainstream dependencies on common targets all publish wheels. You only hit an
sdist for (a) older, low-maintenance pure-Python packages that never uploaded a
wheel, (b) compiled packages on a brand-new Python or uncommon platform before
wheels are published, or (c) the occasional source-only package.

If the lock pins a package with no compatible wheel, the build fails fast and
tells you exactly how to proceed: pre-build the wheel out-of-band with your own
(trusted, ideally isolated) tooling and point `--find-links` at it.

```
# Build wheels once, with isolation/tooling you control:
uv pip wheel -r requirements.txt -w ./wheelhouse   # or: pip wheel ...

# Then build the image from the local wheelhouse:
pymage build --find-links ./wheelhouse -t latest
```

This keeps the escape hatch for the rare cases while keeping `pymage`'s builds
hermetic, reproducible, multi-arch, and free of arbitrary build-time code. In
practice the wheels-only constraint costs very little: a survey of real projects
found **~99.7% of the dependencies apps actually use** are installable, and most
projects build unchanged (see [`docs/wheel-coverage-survey.md`](./docs/wheel-coverage-survey.md)).

### Base image requirements (OS / system libraries)

`pymage` installs Python wheels on top of the specified base image; it does **not** install
OS packages. The base image must already provide everything your dependencies
need at runtime beyond the interpreter and pure-Python code, including:

- the Python interpreter and standard library (matching the lock's `cp` tags);
- shared system libraries that compiled wheels link against (e.g. `libffi`,
  `libssl`, `libstdc++`, `libgomp` for some ML wheels); and
- non-Python runtime tools your app shells out to (e.g. ImageMagick for Wand,
  `ffmpeg`, `git`).

Choose (or build) a base that bundles these. For Debian-style bases that means a
variant with the libraries preinstalled; for Chainguard/Wolfi, compose a base
with the needed `apk` packages. If a dependency needs a system library the base
lacks, the image builds fine but **fails at runtime** -- `pymage`can't add `apt`/`apk`
packages for you.

### Choosing a base image

The base is an input to the build, so it affects reproducibility just like the lock and source do. **Pin it by digest** (e.g. `cgr.dev/chainguard/python@sha256:...`) for stable, reproducible rebuilds.

A floating tag such as `cgr.dev/chainguard/python:latest` works, but be aware:

- it makes the base an *uncontrolled input*, so rebuilds aren't reproducible and may push fresh base layers whenever the base image changes; and
- `:latest` slides forward across Python **minor versions**. Pure-Python wheels keep working (they're matched by `py3` and found via `PYTHONPATH`), but version-specific compiled wheels (`cp312`...) break when the interpreter moves.
- `pymage` will attempt to find wheels available compatible with the base image's Python version, but if that version is new there may not be wheels available yet, and if they are, they will result in new layers and a slower build.

`pymage` detects the base's Python version and uses it automatically, so the `--python` flag is usually unnecessary. You can use it to tell `pymage` to fail a build if the base image's Python version changes.

Detection looks at the `PYTHON_VERSION` env var (for official Python images) and, when that's absent, the `python-X.Y` package in `/etc/apko.json` from the top layer (Chainguard/Wolfi images).

Bases that expose neither signal can't be auto-detected -- pass `--python` explicitly, or pin a base that advertises its version.

## Acknowledgements

The idea of building Python images this way dates back almost ten years, to [**@mattmoor**](http://github.com/mattmoor), [`rules_docker`](https://www.youtube.com/watch?v=lviLZFciDv4) and a project called FTL ("Faster than Light"). This concept was what demystified container images for me ("they're just tarballs and JSON?!!") and changed my whole career, hopefully for the better. This project is just a resurrection of that project for a more civilized time.
