# CSE 490 Project 6

This repository is the course copy of thumbor for CSE 490 Project 6. This section is the course's. The rest of this file is upstream's own AGENTS.md and still applies where it does not conflict with this section.

## Dev install in this course

Use Python 3.12 or newer. Run these commands at the repository root:

    python3 -m venv venv
    source venv/bin/activate
    pip install -U pip setuptools wheel
    pip install -e ".[tests,opencv,extra-libs]"
    python setup.py build_ext -i

The last command builds the C extensions. The server and the tests need them.

Run the test suite with:

    python -m pytest tests -q -p no:cacheprovider -p no:tldr --ignore=tests/detectors --ignore=tests/integration_tests

The gifsicle binary is optional on a laptop. Without it, the tests in tests/engines/test_gif.py fail, and so do tests/handlers/test_base_handler_with_cover.py::test_can_get_image_cover and tests/handlers/test_base_handler_with_different_storage_config.py::test_meta_frame_count_with_gif. Three tests in tests/handlers/test_base_handler_with_result_storage.py load an animated GIF through gifsicle and fail without it too. tests/test_doctor.py::test_get_doctor_output_no_config needs gifsicle, jpegtran and ffmpeg. tests/loaders/test_http_loader.py::HttpCurlTimeoutLoaderTestCase::test_load_with_speed_timeout depends on local timing and can fail on a laptop. Those failures are not defects. The checks on GitHub install those programs, so there every test is expected to pass.

## Course server

Start the server with:

    python -m thumbor.server -p 8888 -c course/thumbor.conf -l error

course/thumbor.conf uses the file loader with course/images as its root. course/images holds the pictures the issues refer to: small.png, example.jpg, sticker.gif, rainbow.png, 940x2.png and hills.png. A URL such as http://127.0.0.1:8888/unsafe/fit-in/300x300/filters:fill(red):no_upscale():format(png)/small.png shows the transform in the browser. course/issues.json lists the issues with the URL each one is seen at.

## Trace and note paths

The build run writes its trace to course/trace-build.jsonl. The review run writes its trace to course/trace-review.jsonl. Your notes on where the issue lives go in course/pile.md. This file is the one owner of those three paths. No other file defines them.

## Conventions

Work on a branch named issue-N, where N is the number of the issue you fix. Never push to main. Never force-push. Every fix adds one new test file under tests/ named tests/test_<something>.py. The new test fails without the fix and passes with it. A fix changes at least two source files under thumbor/. Do not weaken, skip or delete existing tests. Do not open issues, pull requests or comments on the upstream thumbor project.

## Pull request description

The human writes the pull request description by hand. It names the issue. It names every changed source file. It names the AI tools used.

---

# AGENTS.md

thumbor is a smart imaging HTTP service built with Python/Tornado. It resizes,
crops, and applies filters to images on demand via URL parameters. The image
processing pipeline is pluggable: loaders, storages, engines, detectors, and
filters are all swappable.

## Dev Environment Setup

- **Prerequisites:** Python 3.10+, uv, Redis, a C compiler (for native
  extensions). The uv development workflow requires macOS 13+ on macOS.
- Install all dependencies (creates `.venv` with an editable install, test
  tools, and optional imaging libraries):

  ```
  make setup
  ```

  Equivalent to: `uv sync --locked --extra tests --extra all`

- Install pre-commit hooks (run once after setup):

  ```
  make pre-commit
  ```

- Compile C extensions (required before running the server or tests locally):

  ```
  make compile_ext
  ```

## Running the server

```
make run        # debug mode: thumbor -l debug -d -c thumbor/thumbor.conf
make run-prod   # production mode: thumbor -l error -c thumbor/thumbor.conf
```

Other CLI entry points registered by the package:

- `thumbor-url` — compose and sign URLs
- `thumbor-config` — generate a config file
- `thumbor-doctor` — diagnose the local installation

## Running tests

- Full suite (builds extensions, starts Redis, runs unit + integration + lint,
  stops Redis):

  ```
  make test
  ```

- Unit tests only (parallel, uses all CPUs):

  ```
  make unit
  ```

- Integration tests only:

  ```
  make integration_run
  ```

- Coverage report:

  ```
  make coverage
  ```

- Tests require a local Redis on port **6668** (sentinel on **26379**, password
  `hey_you`). Start it with:

  ```
  make redis
  ```

  If you hit "Too many open files", run `ulimit -S -n 2048` first.

- In CI, everything runs inside Docker across a Python 3.10–3.14 matrix via
  GitHub Actions.

## Code style

- **Formatter:** `black` (line length 79, target Python 3.10). Run with `make format`.
- **Linter:** `flake8` (max complexity 20). Run with `make flake`.
- **Imports:** `isort` (multi-line style 3, compatible with black).
- **Static analysis:** `pylint`. Run with `make pylint`.
- Pre-commit runs Black, Flake8, isort, and Pylint through the locked uv
  environment. It also checks trailing whitespace, end-of-file newlines, and
  YAML validity.
- CI validates `uv.lock` and runs the same code-quality checks on every
  push/PR.

## Project structure

```
thumbor/              ← main Python package
  app.py              ← Tornado application
  server.py           ← CLI entry point
  config.py           ← all configuration knobs
  transformer.py      ← image transformation pipeline
  handlers/           ← HTTP handlers (imaging, upload, healthcheck)
  engines/            ← image processing engines (Pillow, GIF, JSON)
  filters/            ← built-in filters (blur, brightness, watermark…)
  detectors/          ← face/feature detectors (OpenCV Haar cascades)
  loaders/            ← image source loaders (http, file, strict)
  storages/           ← image storages (file, mixed, no-op)
  result_storages/    ← result caching
  url_signers/        ← HMAC URL signing
  ext/filters/        ← C extension modules for performance-critical filters
tests/                ← unit tests (pytest + pytest-xdist)
integration_tests/    ← integration tests
docs/                 ← Sphinx documentation
docker/               ← Docker files
.github/workflows/    ← CI: build, lint, release, CodeQL, dependabot
Makefile              ← primary task runner (use this for all common tasks)
```

## Contribution guidelines

- Bug fixes for the v6 stable branch go to `fixes/6.7.x`; new features go to `master`.
- Merge `master` into your branch before opening a PR.
- Every change must include tests. Run `make unit` to verify.
- Run `make flake isort pylint` before committing (or rely on the pre-commit
  hooks).
- Do not add a `thumbor.key` file with real secrets; the repo-root `thumbor.key`
  is for development only.

## Security considerations

- **`SECURITY_KEY`** defaults to `"MY_SECURE_KEY"`. Always override this in production.
- **`ALLOW_UNSAFE_URL`** defaults to `True` (bypasses URL signing). Set to
  `False` in production.
- **`ALLOWED_SOURCES`** defaults to `[]` (all sources allowed). Restrict to
  known domains in production using regex patterns.
- **`MAX_PIXELS`** defaults to 75 megapixels; keep it set to guard against
  decompression bombs.
- CodeQL scans Python and C/C++ code on every push to `master` and weekly via
  GitHub Actions.
