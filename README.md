# hosttrace: application tracing with static/dynamic binary instrumentation

## Dependencies

- [DynInst](https://github.com/dyninst/dyninst) for dynamic or static instrumentation
- [Julia](https://julialang.org/) for merging perfetto traces

## Installing DynInst

The easiest way to install Dyninst is via spack

```shell
git clone https://github.com/spack/spack.git
source ./spack/share/spack/setup-env.sh
spack compiler find
spack external find
spack install dyninst
spack load -r dyninst
```

## Installing Julia

Julia is available via Linux package managers or may be available via a module. Debian-based distributions such as Ubuntu can run (as a super-user):

```shell
apt-get install julia
```

Once Julia is installed, install the necessary packages (this operation only needs to be performed once):

```shell
julia -e 'using Pkg; for name in ["JSON", "DataFrames", "Dates", "CSV", "Chain", "PrettyTables"]; Pkg.add(name); end'
```

## Installing hosttrace

```shell
HOSTTRACE_ROOT=${HOME}/sw/hosttrace
git clone https://github.com/AARInternal/hosttrace-dyninst.git
cmake -B build-hosttrace -DHOSTTRACE_USE_MPI=ON -DCMAKE_INSTALL_PREFIX=${HOSTTRACE_ROOT} hosttrace-dyninst
cmake --build build-hosttrace --target all --parallel 8
cmake --build build-hosttrace --target install
export PATH=${HOSTTRACE_ROOT}/bin:${PATH}
export LD_LIBRARY_PATH=${HOSTTRACE_ROOT}/lib64:${HOSTTRACE_ROOT}/lib:${LD_LIBRARY_PATH}
```

## Using Hosttrace Executable

```shell
hosttrace --help
hosttrace <hosttrace-options> -- <exe-or-library> <exe-options>
```

## Hosttrace Library Environment Settings

| Environment Variable        | Default Value                 | Description                                                                      |
|-----------------------------|-------------------------------|----------------------------------------------------------------------------------|
| `HOSTTRACE_DEBUG`           | `false`                       | Enable debugging statements                                                      |
| `HOSTTRACE_USE_PERFETTO`    | `true`                        | Collect profiling data via perfetto                                              |
| `HOSTTRACE_USE_TIMEMORY`    | `false`                       | Collection profiling data via timemory                                           |
| `HOSTTRACE_SAMPLE_RATE`     | `1`                           | Invoke perfetto and/or timemory once every N function calls                      |
| `HOSTTRACE_USE_MPI`         | `true`                        | Label perfetto output files via rank instead of PID                              |
| `HOSTTRACE_OUTPUT_FILE`     | `perfetto-trace.%rank%.proto` | Output file for perfetto (may use `%pid`)                                        |
| `HOSTTRACE_BACKEND`         | `"inprocess"`                 | Configure perfetto to use either "inprocess" data management, "system", or "all" |
| `HOSTTRACE_COMPONENTS`      | `"wall_clock"`                | Timemory components to activate when enabled                                     |
| `HOSTTRACE_SHMEM_SIZE_HINT` | `40960`                       | Hint for perfetto shared memory buffer                                           |
| `HOSTTRACE_BUFFER_SIZE_KB`  | `1024000`                     | Maximum amount of memory perfetto will use to collect data in-process            |
| `TIMEMORY_TIME_OUTPUT`      | `true`                        | Create unique output subdirectory with date and launch time                      |

### Example Hosttrace Instrumentation

#### Binary Rewrite

Rewrite the text section of an executable or library with instrumentation:

```shell
hosttrace -o app.inst -- /path/to/app
```

In binary rewrite mode, if you also want instrumentation in the linked libraries, you must also rewrite those libraries.
Example of rewriting the functions starting with `"hip"` with instrumentation in the amdhip64 library:

```shell
mkdir -p ./lib
hosttrace -R '^hip' -o ./lib/libamdhip64.so.4 -- /opt/rocm/lib/libamdhip64.so.4
export LD_LIBRARY_PATH=${PWD}/lib:${LD_LIBRARY_PATH}
```

> NOTE: Verify via `ldd` that your executable will load the instrumented library -- if you built your executable with
> an RPATH to the original library's directory, then prefixing `LD_LIBRARY_PATH` will have no effect.

Once you have rewritten your executable and/or libraries with instrumentation, you can just run the (instrumented) executable
or exectuable which loads the instrumented libraries normally, e.g.:

```shell
./app.inst
```

If you want to re-define certain settings to new default in a binary rewrite, use the `--env` option. This `hosttrace` option
will set the environment variable to the given value but will not override it. E.g. the default value of `HOSTTRACE_BUFFER_SIZE_KB`
is 1024000 KB (1 GiB):

```shell
# buffer size defaults to 1024000
hosttrace -o app.inst -- /path/to/app
./app.inst
```

Passing `--env HOSTTRACE_BUFFER_SIZE_KB=5120000` will change the default value in `app.inst` to 5120000 KiB (5 GiB):

```shell
# defaults to 5 GiB buffer size
hosttrace -o app.inst --env HOSTTRACE_BUFFER_SIZE_KB=5120000 -- /path/to/app
./app.inst
```

```shell
# override default 5 GiB buffer size to 200 MB
export HOSTTRACE_BUFFER_SIZE_KB=200000
./app.inst
```

#### Runtime Instrumentation

Runtime instrumentation will not only instrument the text section of the executable but also the text sections of the
linked libraries. Thus, it may be useful to exclude those libraries via the `-ME` (module exclude) regex option.

```shell
hosttrace -- /path/to/app
hosttrace -ME '^(libhsa-runtime64|libz\\.so)' -- /path/to/app
hosttrace -E 'rocr::atomic|rocr::core|rocr::HSA' --  /path/to/app
```

## Miscellaneous Features and Caveats

- You may need to increase the default perfetto buffer size (1 GiB) to capture all the information
  - E.g. `export HOSTTRACE_BUFFER_SIZE_KB=10240000` increases the buffer size to 10 GiB
- The hosttrace library has various setting which can be configured via environment variables, you can
  configure these settings to custom defaults with the hosttrace command-line tool via the `--env` option
  - E.g. to default to a buffer size of 5 GB, use `--env HOSTTRACE_BUFFER_SIZE_KB=5120000`
  - This is particularly useful in binary rewrite mode
- Perfetto tooling is enabled by default
- Timemory tooling is disabled by default
- Enabling/disabling one of the aformentioned tools but not specifying enabling/disable the other will assume the inverse of the other's enabled state, e.g.
  - `HOSTTRACE_USE_PERFETTO=OFF` yields the same result `HOSTTRACE_USE_TIMEMORY=ON`
  - `HOSTTRACE_USE_PERFETTO=ON` yields the same result as `HOSTTRACE_USE_TIMEMORY=OFF`
  - In order to enable _both_ timemory and perfetto, set both `HOSTTRACE_USE_TIMEMORY=ON` and `HOSTTRACE_USE_PERFETTO=ON`
  - Setting `HOSTTRACE_USE_TIMEMORY=OFF` and `HOSTTRACE_USE_PERFETTO=OFF` will disable all instrumentation
- Use `timemory-avail -S` to view the various settings for timemory
- Set `HOSTTRACE_COMPONENTS="<comma-delimited-list-of-component-name>"` to control which components timemory collects
  - The list of components and their descriptions can be viewed via `timemory-avail -Cd`
  - The list of components and their string identifiers can be view via `timemory-avail -Cbs`
- You can filter any `timemory-avail` results via `-r <regex> -hl`

## Hosttrace Output

`hosttrace` will create an output directory named `hosttrace-<EXE_NAME>-output`, e.g. if your executable
is named `app.inst`, the output directory will be `hosttrace-app.inst-output`. Depending on whether
`TIMEMORY_TIME_OUTPUT=ON` (the default when perfetto is enabled), there will be a subdirectory with the date and time,
e.g. `2021-09-02_01.03_PM`. Within this directory, all perfetto files will be named `perfetto-trace.<PID>.proto` or
when `HOSTTRACE_USE_MPI=ON`, `perfetto-trace.<RANK>.proto` (assuming hosttrace was built with MPI support).

You can explicitly control the output path and naming scheme of the files via the `HOSTTRACE_OUTPUT_FILE` environment
variable. The special character sequences `%pid%` and `%rank%` will be replaced with the PID or MPI rank, respectively.

## Merging the traces from rocprof and hosttrace

> NOTE: Using `rocprof` externally is deprecated. The current version has built-in support for
> recording the GPU activity and HIP API calls. If you want to use an external rocprof, either
> configure CMake with `-DHOSTTRACE_USE_ROCTRACER=OFF` or explicitly set `TIMEMORY_ROCTRACER_ENABLED=OFF` in the
> environment.

Use the `hosttrace-merge.jl` Julia script to merge rocprof and perfetto traces.

```shell
export TIMEMORY_ROCTRACER_ENABLED=OFF
rocprof --hip-trace --roctx-trace --stats ./app.inst
hosttrace-merge.jl results.json hosttrace-app.inst-output/2021-09-02_01.03_PM/*.proto
```

## Use Perfetto tracing with System Backend

In a separate window run:

```shell
pkill traced
traced --background
perfetto --out ./htrace.out --txt -c ${HOSTTRACE_ROOT}/share/roctrace.cfg
```

then in the window running the application, configure the hosttrace instrumentation to use the system backend:

```shell
export HOSTTRACE_BACKEND_SYSTEM=1
```

for the merge use the `htrace.out`:

```shell
hosttrace-merge.jl results.json htrace.out
```