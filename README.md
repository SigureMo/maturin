# Maturin

_formerly pyo3-pack_

[![Actions Status](https://img.shields.io/github/actions/workflow/status/PyO3/maturin/test.yml?branch=main&logo=github&style=flat-square)](https://github.com/PyO3/maturin/actions)
[![FreeBSD](https://img.shields.io/cirrus/github/PyO3/maturin/main?logo=CircleCI&style=flat-square)](https://cirrus-ci.com/github/PyO3/maturin)
[![Bors enabled](https://bors.tech/images/badge_small.svg)](https://app.bors.tech/repositories/55651)
[![Crates.io](https://img.shields.io/crates/v/maturin.svg?logo=rust&style=flat-square)](https://crates.io/crates/maturin)
[![PyPI](https://img.shields.io/pypi/v/maturin.svg?logo=python&style=flat-square)](https://pypi.org/project/maturin)
[![Maturin User Guide](https://img.shields.io/badge/user-guide-brightgreen?logo=readthedocs&style=flat-square)](https://maturin.rs)
[![Chat on Gitter](https://img.shields.io/gitter/room/nwjs/nw.js.svg?logo=gitter&style=flat-square)](https://gitter.im/PyO3/Lobby)

Build and publish crates with pyo3, rust-cpython, cffi and uniffi bindings as well as rust binaries as python packages.

This project is meant as a zero configuration replacement for [setuptools-rust](https://github.com/PyO3/setuptools-rust) and [milksnake](https://github.com/getsentry/milksnake).
It supports building wheels for python 3.5+ on windows, linux, mac and freebsd, can upload them to [pypi](https://pypi.org/) and has basic pypy support.

Check out the [User Guide](https://maturin.rs/)!

## Usage

You can either download binaries from the [latest release](https://github.com/PyO3/maturin/releases/latest) or install it with pip:

```shell
pip install maturin
```

There are four main commands:

 * `maturin new` creates a new cargo project with maturin configured.
 * `maturin publish` builds the crate into python packages and publishes them to pypi.
 * `maturin build` builds the wheels and stores them in a folder (`target/wheels` by default), but doesn't upload them. It's possible to upload those with [twine](https://github.com/pypa/twine) or `maturin upload`.
 * `maturin develop` builds the crate and installs it as a python module directly in the current virtualenv. Note that while `maturin develop` is faster, it doesn't support all the feature that running `pip install` after `maturin build` supports.

`pyo3` and `rust-cpython` bindings are automatically detected. For cffi or binaries, you need to pass `-b cffi` or `-b bin`.
maturin doesn't need extra configuration files and doesn't clash with an existing setuptools-rust or milksnake configuration.
You can even integrate it with testing tools such as [tox](https://tox.readthedocs.io/en/latest/).
There are examples for the different bindings in the `test-crates` folder.

The name of the package will be the name of the cargo project, i.e. the name field in the `[package]` section of `Cargo.toml`.
The name of the module, which you are using when importing, will be the `name` value in the `[lib]` section (which defaults to the name of the package). For binaries, it's simply the name of the binary generated by cargo.

## Python packaging basics

Python packages come in two formats:
A built form called wheel and source distributions (sdist), both of which are archives.
A wheel can be compatible with any python version, interpreter (cpython and pypy, mainly), operating system and hardware architecture (for pure python wheels),
can be limited to a specific platform and architecture (e.g. when using ctypes or cffi) or to a specific python interpreter and version on a specific architecture and operating system (e.g. with pyo3 and rust-cpython).

When using `pip install` on a package, pip tries to find a matching wheel and install that. If it doesn't find one, it downloads the source distribution and builds a wheel for the current platform,
which requires the right compilers to be installed. Installing a wheel is much faster than installing a source distribution as building wheels is generally slow.

When you publish a package to be installable with `pip install`, you upload it to [pypi](https://pypi.org/), the official package repository.
For testing, you can use [test pypi](https://test.pypi.org/) instead, which you can use with `pip install --index-url https://test.pypi.org/simple/`.
Note that for publishing for linux, [you need to use the manylinux docker container](#manylinux-and-auditwheel), while for publishing from your repository you can use the [PyO3/maturin-action github action](https://github.com/PyO3/maturin-action).

## pyo3 and rust-cpython

For pyo3 and rust-cpython, maturin can only build packages for installed python versions. On linux and mac, all python versions in `PATH` are used.
If you don't set your own interpreters with `-i`, a heuristic is used to search for python installations.
On windows all versions from the python launcher (which is installed by default by the python.org installer) and all conda environments except base are used. You can check which versions are picked up with the `list-python` subcommand.

pyo3 will set the used python interpreter in the environment variable `PYTHON_SYS_EXECUTABLE`, which can be used from custom build scripts. Maturin can build and upload wheels for pypy with pyo3, even though only pypy3.7-7.3 on linux is tested.

## Cffi

Cffi wheels are compatible with all python versions including pypy. If `cffi` isn't installed and python is running inside a virtualenv, maturin will install it, otherwise you have to install it yourself (`pip install cffi`).

maturin uses cbindgen to generate a header file, which can be customized by configuring cbindgen through a `cbindgen.toml` file inside your project root. Alternatively you can use a build script that writes a header file to `$PROJECT_ROOT/target/header.h`.

Based on the header file maturin generates a module which exports an `ffi` and a `lib` object.

<details>
<summary>Example of a custom build script</summary>

```rust
use cbindgen;
use std::env;
use std::path::Path;

fn main() {
    let crate_dir = env::var("CARGO_MANIFEST_DIR").unwrap();

    let bindings = cbindgen::Builder::new()
        .with_no_includes()
        .with_language(cbindgen::Language::C)
        .with_crate(crate_dir)
        .generate()
        .unwrap();
    bindings.write_to_file(Path::new("target").join("header.h"));
}
```

</details>

## uniffi

uniffi bindings use [uniffi-rs](https://mozilla.github.io/uniffi-rs/) to generate Python `ctypes` bindings
from an interface definition file. uniffi wheels are compatible with all python versions including pypy.

## Mixed rust/python projects

To create a mixed rust/python project, create a folder with your module name (i.e. `lib.name` in Cargo.toml) next to your Cargo.toml and add your python sources there:

```
my-project
├── Cargo.toml
├── my_project
│   ├── __init__.py
│   └── bar.py
├── pyproject.toml
├── README.md
└── src
    └── lib.rs
```

You can specify a different python source directory in `pyproject.toml` by setting `tool.maturin.python-source`, for example

**pyproject.toml**

```toml
[tool.maturin]
python-source = "python"
```

then the project structure would look like this:

```
my-project
├── Cargo.toml
├── python
│   └── my_project
│       ├── __init__.py
│       └── bar.py
├── pyproject.toml
├── README.md
└── src
    └── lib.rs
```

> **Note**
>
> This structure is recommended to avoid [a common `ImportError` pitfall](https://github.com/PyO3/maturin/issues/490)

maturin will add the native extension as a module in your python folder. When using develop, maturin will copy the native library and for cffi also the glue code to your python folder. You should add those files to your gitignore.

With cffi you can do `from .my_project import lib` and then use `lib.my_native_function`, with pyo3/rust-cpython you can directly `from .my_project import my_native_function`.

Example layout with pyo3 after `maturin develop`:

```
my-project
├── Cargo.toml
├── my_project
│   ├── __init__.py
│   ├── bar.py
│   └── my_project.cpython-36m-x86_64-linux-gnu.so
├── README.md
└── src
    └── lib.rs
```

## Python metadata

maturin supports [PEP 621](https://www.python.org/dev/peps/pep-0621/), you can specify python package metadata in `pyproject.toml`.
maturin merges metadata from `Cargo.toml` and `pyproject.toml`, `pyproject.toml` take precedence over `Cargo.toml`.

To specify python dependencies, add a list `dependencies` in a `[project]` section in the `pyproject.toml`. This list is equivalent to `install_requires` in setuptools:

```toml
[project]
name = "my-project"
dependencies = ["flask~=1.1.0", "toml==0.10.0"]
```

Pip allows adding so called console scripts, which are shell commands that execute some function in you program. You can add console scripts in a section `[project.scripts]`.
The keys are the script names while the values are the path to the function in the format `some.module.path:class.function`, where the `class` part is optional. The function is called with no arguments. Example:

```toml
[project.scripts]
get_42 = "my_project:DummyClass.get_42"
```

You can also specify [trove classifiers](https://pypi.org/classifiers/) in your `pyproject.toml` under `project.classifiers`:

```toml
[project]
name = "my-project"
classifiers = ["Programming Language :: Python"]
```

## Source distribution

maturin supports building through `pyproject.toml`. To use it, create a `pyproject.toml` next to your `Cargo.toml` with the following content:

```toml
[build-system]
requires = ["maturin>=0.14,<0.15"]
build-backend = "maturin"
```

If a `pyproject.toml` with a `[build-system]` entry is present, maturin can build a source distribution of your package when `--sdist` is specified.
The source distribution will contain the same files as `cargo package`. To only build a source distribution, pass `--interpreter` without any values.

You can then e.g. install your package with `pip install .`. With `pip install . -v` you can see the output of cargo and maturin.

You can use the options `compatibility`, `skip-auditwheel`, `bindings`, `strip` and common Cargo build options such as `features` under `[tool.maturin]` the same way you would when running maturin directly.
The `bindings` key is required for cffi and bin projects as those can't be automatically detected. Currently, all builds are in release mode (see [this thread](https://discuss.python.org/t/pep-517-debug-vs-release-builds/1924) for details).

For a non-manylinux build with cffi bindings you could use the following:

```toml
[build-system]
requires = ["maturin>=0.14,<0.15"]
build-backend = "maturin"

[tool.maturin]
bindings = "cffi"
compatibility = "linux"
```

`manylinux` option is also accepted as an alias of `compatibility` for backward compatibility with old version of maturin.

To include arbitrary files in the sdist for use during compilation specify `include` as an array of `path` globs with `format` set to `sdist`:

```toml
[tool.maturin]
include = [{ path = "path/**/*", format = "sdist" }]
```

There's a `maturin sdist` command for only building a source distribution as workaround for [pypa/pip#6041](https://github.com/pypa/pip/issues/6041).

## Manylinux and auditwheel

For portability reasons, native python modules on linux must only dynamically link a set of very few libraries which are installed basically everywhere, hence the name manylinux.
The pypa offers special docker images and a tool called [auditwheel](https://github.com/pypa/auditwheel/) to ensure compliance with the [manylinux rules](https://peps.python.org/pep-0599/#the-manylinux2014-policy).
If you want to publish widely usable wheels for linux pypi, **you need to use a manylinux docker image**.

The Rust compiler since version 1.64 [requires at least glibc 2.17](https://blog.rust-lang.org/2022/08/01/Increasing-glibc-kernel-requirements.html), so you need to use at least manylinux2014.
For publishing, we recommend enforcing the same manylinux version as the image with the manylinux flag, e.g. use `--manylinux 2014` if you are building in `quay.io/pypa/manylinux2014_x86_64`.
The [PyO3/maturin-action](https://github.com/PyO3/maturin-action) github action already takes care of this if you set e.g. `manylinux: 2014`.

maturin contains a reimplementation of auditwheel automatically checks the generated library and gives the wheel the proper.
If your system's glibc is too new or you link other shared libraries, it will assign the `linux` tag.
You can also manually disable those checks and directly use native linux target with `--manylinux off`.

For full manylinux compliance you need to compile in a CentOS docker container. The [pyo3/maturin](https://ghcr.io/pyo3/maturin) image is based on the manylinux2014 image,
and passes arguments to the `maturin` binary. You can use it like this:

```
docker run --rm -v $(pwd):/io ghcr.io/pyo3/maturin build --release  # or other maturin arguments
```

Note that this image is very basic and only contains python, maturin and stable rust. If you need additional tools, you can run commands inside the manylinux container.
See [konstin/complex-manylinux-maturin-docker](https://github.com/konstin/complex-manylinux-maturin-docker) for a small educational example or [nanoporetech/fast-ctc-decode](https://github.com/nanoporetech/fast-ctc-decode/blob/b226ea0f2b2f4f474eff47349703d57d2ea4801b/.github/workflows/publish.yml) for a real world setup.

maturin itself is manylinux compliant when compiled for the musl target.

## Examples

* [ballista-python](https://github.com/apache/arrow-ballista-python) - A Python library that binds to Apache Arrow distributed query engine Ballista
* [connector-x](https://github.com/sfu-db/connector-x/tree/main/connectorx-python) - ConnectorX enables you to load data from databases into Python in the fastest and most memory efficient way
* [datafusion-python](https://github.com/apache/arrow-datafusion-python) - a Python library that binds to Apache Arrow in-memory query engine DataFusion
* [deltalake-python](https://github.com/delta-io/delta-rs/tree/main/python) - Native Delta Lake Python binding based on delta-rs with Pandas integration
* [orjson](https://github.com/ijl/orjson) - A fast, correct JSON library for Python
* [polars](https://github.com/pola-rs/polars/tree/master/py-polars) - Fast multi-threaded DataFrame library in Rust | Python | Node.js
* [pydantic-core](https://github.com/pydantic/pydantic-core) - Core validation logic for pydantic written in Rust
* [pyrus-cramjam](https://github.com/milesgranger/pyrus-cramjam) - Thin Python wrapper to de/compression algorithms in Rust
* [pyxel](https://github.com/kitao/pyxel) - A retro game engine for Python
* [roapi](https://github.com/roapi/roapi) - ROAPI automatically spins up read-only APIs for static datasets without requiring you to write a single line of code
* [robyn](https://github.com/sansyrox/robyn) -  A fast and extensible async python web server with a Rust runtime
* [ruff](https://github.com/charliermarsh/ruff) - An extremely fast Python linter, written in Rust
* [tantivy-py](https://github.com/quickwit-oss/tantivy-py) - Python bindings for Tantivy
* [watchfiles](https://github.com/samuelcolvin/watchfiles) - Simple, modern and high performance file watching and code reload in python
* [wonnx](https://github.com/webonnx/wonnx/tree/master/wonnx-py) - Wonnx is a GPU-accelerated ONNX inference run-time written 100% in Rust

## Contributing

Everyone is welcomed to contribute to maturin! There are many ways to support the project, such as:

- help maturin users with issues on GitHub and Gitter
- improve documentation
- write features and bugfixes
- publish blogs and examples of how to use maturin

Our [contributing notes](https://github.com/PyO3/maturin/blob/main/guide/src/contributing.md) have more resources if you wish to volunteer time for maturin and are searching where to start.

If you don't have time to contribute yourself but still wish to support the project's future success, some of our maintainers have GitHub sponsorship pages:

- [messense](https://github.com/sponsors/messense)

## License

Licensed under either of:

 * Apache License, Version 2.0, ([LICENSE-APACHE](https://github.com/PyO3/maturin/blob/main/license-apache) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license ([LICENSE-MIT](https://github.com/PyO3/maturin/blob/main/license-mit) or http://opensource.org/licenses/MIT)

at your option.
