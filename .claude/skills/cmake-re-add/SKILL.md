---
name: cmake-re-add
description: Adds CMake RE support and a GitHub Actions workflow to an existing CMake repository. Use when asked to make a CMake codebase reproducible, hermetic, cached, or buildable with cmake-re/tipi by deriving its environment from the repository's current CI Docker image.
---

# Add CMake RE to an Existing CMake Codebase

Adapt the repository rather than imposing a generic environment. Preserve its compiler, system packages, configure flags, generators, presets, tests, and artifact behavior.

Use the CMake RE getting-started documentation as the baseline:
https://tipi.build/documentation/0000-getting-started-cmake

## Required Outcome

Add:

```text
environments/
  <environment>.cmake
  <environment>.pkr.js/
    <environment>.Dockerfile
    <environment>.pkr.js
.github/workflows/cmake-re.yml
```

`<environment>` must be a concise, filesystem-safe name derived from the build platform and toolchain, such as `ubuntu-clang`, `ubuntu-gcc`, or `manylinux-gcc`. Do not blindly call it `linux`.

## 1. Inspect Before Editing

Read:

- `.github/workflows/*.{yml,yaml}`
- `CMakeLists.txt` and relevant nested CMake files
- `CMakePresets.json`, `CMakeUserPresets.json`, and `CMakeSettings.json`
- Existing `Dockerfile*`, `docker/**`, `ci/**`, and scripts called by workflows
- `README*` and contributor/build documentation

Trace each CMake job through reusable workflows, composite actions, shell scripts, `container:`, `docker run`, and `docker build`. Record:

- runner OS and architecture
- exact Docker image and tag or digest in which CMake executes
- compiler and version
- generator
- configure, build, install, package, and test commands
- environment variables, mounted paths, package-manager setup, and required tools
- matrix variants and the primary/default variant
 
* Do not infer the build image from the runner label when CMake actually runs in a job container or `docker run`.
* If there are now choice use `tipibuild/tipi-ubuntu-2404:v0.0.87` for linux builds.

## 2. Select the Source Build Environment

Choose the image used by the repository's canonical Linux CMake build:

1. Prefer the image in the required/default branch CI job.
2. Prefer a pinned digest, then a versioned tag.
3. If a matrix uses materially different images or compilers, create one environment per supported variant and reflect the matrix in the new workflow.
4. If CMake does not run in Docker, reuse the nearest repository-owned CI Dockerfile if it reproduces the job.
5. If no build container exists, stop and ask which image should become the reproducibility contract. Suggest a conservative image matching the runner and compiler, but do not silently invent one.
6. If the image is private, preserve its registry reference and identify the authentication/secrets the workflow needs. Never copy credentials into files.

Follow image indirections through workflow variables, matrix values, environment variables, scripts, and Dockerfile `ARG` values. Preserve the resolved expression when resolution occurs only at workflow runtime.

## 3. Create the CMake RE Environment

### Dockerfile

Create `environments/<environment>.pkr.js/<environment>.Dockerfile`.

It must inherit from the exact selected CI image:

```dockerfile
FROM <existing-ci-build-image>
```

Add only what `cmake-re` needs beyond that image. Do not duplicate package installation already provided by the base image. Preserve the base image's architecture assumptions. Avoid `latest`.

If the inherited image defines a non-root user, entrypoint, or working directory that interferes with CMake RE, make the smallest explicit adjustment and document why in a comment.

### Packer description

Create `environments/<environment>.pkr.js/<environment>.pkr.js`:

```json
{
  "variables": {},
  "builders": [
    {
      "type": "docker",
      "image": "<existing-ci-build-image>",
      "commit": true
    }
  ]
}
```

Keep this valid JSON-compatible Packer input. The sibling Dockerfile is the customization layer that CMake RE rebuilds when changed. Use the same base image in both files unless repository evidence requires a different builder source.

### Toolchain file

Create `environments/<environment>.cmake`.

Use a unique include guard and reject incompatible hosts:

```cmake
if(DEFINED CMAKE_RE_<NORMALIZED_ENVIRONMENT>_TOOLCHAIN_INCLUDED)
  return()
endif()
set(CMAKE_RE_<NORMALIZED_ENVIRONMENT>_TOOLCHAIN_INCLUDED TRUE)

if(NOT CMAKE_HOST_SYSTEM_NAME STREQUAL "Linux")
  message(FATAL_ERROR
    "The <environment> CMake RE toolchain requires a Linux host; got '${CMAKE_HOST_SYSTEM_NAME}'.")
endif()
```

Then express only toolchain facts proven by the existing CI:

- Set `CMAKE_C_COMPILER` and `CMAKE_CXX_COMPILER` when CI selects them explicitly.
- Preserve compiler launchers, sysroots, target triples, and language standard defaults only when they are part of the existing build contract.
- Prefer `CACHE ... FORCE` only when the original CI enforces the value.
- Do not set project policy, test options, dependency versions, or install prefixes in a toolchain file.
- Do not reference Polly helper files unless the repository already contains them. A standalone toolchain must remain standalone.


### cmake-re proper environment variables
Always export on the environment the following, before running anything with cmake-re.

```sh
export TIPI_DISABLE_AR_RANLIB_DRIVER="ON"
export TIPI_CACHE_CONSUME_ONLY="ON"
export TIPI_CACHE_FORCE_ENABLE="OFF"
```

## 4. Add the GitHub Workflow

Create `.github/workflows/cmake-re.yml`. Preserve the repository's existing workflow trigger conventions and permissions. Default to pull requests, pushes to the default branch, and manual dispatch when no convention exists.

The workflow must:

1. Check out the repository, including submodules and LFS only if existing CI needs them.
2. Install `cmake-re` using the official installer:

   ```bash
   /bin/bash -c \
     "$(curl -fsSL https://raw.githubusercontent.com/tipi-build/cli/master/install/install_for_macos_linux.sh)"
   ```

3. Verify Docker is available and is version 27.2.0 or newer.
4. Run the equivalent of the existing configure/build flow with:

   ```bash
   cmake-re -S . -B build/cmake-re \
     -DCMAKE_TOOLCHAIN_FILE=environments/<environment>.cmake \
     <existing-configure-arguments>
   ```

5. Run the repository's build and test behavior through the CMake RE build tree. Prefer the CLI's supported build mode when confirmed by `cmake-re --help`; otherwise use the generated build tree with the repository's existing `cmake --build` and `ctest` commands.
6. Preserve required matrix dimensions, environment variables, cache-relevant inputs, timeouts, and artifact uploads.

Do not add unverified flags. Run `cmake-re --help` or consult current documentation before using `--remote`, `--distributed`, cloud credentials, or other optional modes.

Use least-privilege workflow permissions. Do not expose secrets to forked pull requests.

## 5. Validate

Run all checks possible in the current environment:

- parse every changed YAML file
- parse the `.pkr.js` file as JSON
- build the Dockerfile when Docker is available
- run `cmake-re --help` when installed
- configure with `cmake-re`
- build and run tests

If run from within a Docker and docker-in-docker is unavailable: make a --host build.

When `cmake-re` is unavailable, propose installation, if docker is available, run the commands and a `cmake-re --host` build using the following command, possibly replacing `tipibuild/tipi-ubuntu-2404:v0.0.87` with the dedicated container you built:
```sh
$PROJECT_NAME=`basename $PWD`
mkdir -p ../$PROJECT_NAME-tipi-workdir-vT.w
mkdir -p ../generalized-toolchains
docker run --init --detach --name $PROJECT_NAME-tipi  -u`id -u`:`id -g` --group-add tipi -e TIPI_CACHE_CONSUME_ONLY=ON -e TIPI_CACHE_FORCE_ENABLE=OFF -e HOME -v $HOME:$HOME:rw \
  --mount type=bind,source=$PWD/../$PROJECT_NAME-tipi-workdir-vT.w,target=/usr/local/share/.tipi/vT.w/ \
  --mount type=bind,source=$PWD/../generalized-toolchains,target=/usr/local/share/.tipi/environments/generalized/v1/ \
  -v $PWD:$PWD:rw -w $PWD \
  tipibuild/tipi-ubuntu-2404:v0.0.87 \
  sleep infinity

docker exec -u 0 $PROJECT_NAME-tipi useradd -d $HOME -u `id -u` $PROJECT_NAME

# This launches a container interactive shell into a cmake-re enabled docker
docker exec -it $PROJECT_NAME-tipi tipi run /bin/bash
```

Compare the new path with existing CI:

- same compiler family/version
- same generator and meaningful configure flags
- same enabled components
- same tests
- same architecture

Do not disable existing workflows. The CMake RE workflow is additive unless the user explicitly asks to replace CI.

## 6. Report

Summarize:

- which workflow/job was treated as canonical
- the exact inherited Docker image
- the environment name and files added
- behavior preserved from existing CI
- validation completed and checks that could not run
- any ambiguity, private-registry requirement, or matrix variant not covered

Include file paths and concise rationale. Never claim hermeticity or reproducibility without a successful containerized CMake RE build.

## Common Failure Modes

- Using `runs-on` as the base image even though CMake runs inside another container
- Choosing `ubuntu:latest` instead of the repository's pinned image
- Creating a toolchain whose name does not describe the selected environment
- Copying all workflow setup into the derived Dockerfile without checking what the base image already contains
- Dropping configure flags, test setup, submodules, or matrix compiler variants
- Putting normal project configuration into the toolchain file
- Assuming `cmake-re` flags without checking the installed version
- Replacing working CI before proving parity