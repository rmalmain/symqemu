# SymQEMU

This is SymQEMU, a binary-only symbolic executor based on QEMU and SymCC. It
currently extends QEMU 8.1 and works with the most recent version of SymCC.
(See README.orig for QEMU's original README file.)
A separate branch is available for the [old version of SymQEMU based on QEMU 4.1.1](https://github.com/eurecom-s3/symqemu/tree/4.1.1) we don't expect much 
changes to happen there, but PR may be accepted.

## How to build

SymQEMU requires [SymCC](https://github.com/eurecom-s3/symcc), so please
download and build SymCC first. For best results, configure it with the QSYM
backend as explained in the README. For the impatient, here's a quick summary of
the required steps that may or may not work on your system:

``` shell
$ git clone https://github.com/eurecom-s3/symcc.git
$ cd symcc
$ git submodule update --init
$ mkdir build
$ cd build
$ cmake -G Ninja -DQSYM_BACKEND=ON ..
$ ninja
```

Next, make sure that QEMU's build dependencies are installed. Most package
managers provide a command to get them, e.g., `apt build-dep qemu` on Debian and
Ubuntu, or `dnf builddep qemu` on Fedora and CentOS.

We've extended QEMU's configuration script to accept pointers to SymCC's source
and binaries. The following invocation is known to work on Debian 10, Arch and
Fedora 33:

``` shell
$ mkdir build
$ cd build
$ ../configure                                                    \
      --audio-drv-list=                                           \
      --disable-sdl                                               \
      --disable-gtk                                               \
      --disable-vte                                               \
      --disable-opengl                                            \
      --disable-virglrenderer                                     \
      --disable-werror                                            \
      --target-list=x86_64-linux-user                             \
      --symcc-source=../symcc                                     \
      --symcc-build=../symcc/build
$ make -j
```

This will build a relatively stripped-down emulator targeting 64-bit x86
binaries. We also have experimental support for AARCH64. Working with 32-bit
target architectures is possible in principle but will require a bit of work
because the current implementation assumes that we can pass around host pointers
in guest registers.

## Running SymQEMU

If you built SymQEMU as described above, the binary will be in
`x86_64-linux-user/symqemu-x86_64`. For a quick test, try the following:

``` shell
$ mkdir /tmp/output
$ echo test | x86_64-linux-user/qemu-x86_64 /bin/cat -t -
This is SymCC running with the QSYM backend
[STAT] SMT: { "solving_time": 0, "total_time": 51010 }
[STAT] SMT: { "solving_time": 523 }
[INFO] New testcase: /tmp/output/000000
...
```

This runs your system's `/bin/cat` with options that make it inspect each
character on standard input to check whether or not it's in the non-printable
range. In `/tmp/output`, the default location for test cases generated by
SymQEMU, you'll find versions of the input (i.e., "test") containing
non-printable characters in various positions.

You can have a quick look at the results with the following one-liner:
``` shell
for  i in /tmp/output/00000* ; do ; od -A x -t x1z  $i ; done
```

This is a very basic use of symbolic execution. See SymCC's documentation for
more advanced scenarios. Since SymQEMU is based on it, it understands all the
same
[settings](https://github.com/eurecom-s3/symcc/blob/master/docs/Configuration.txt),
and you can even run SymQEMU with `symcc_fuzzing_helper` for [hybrid fuzzing](https://github.com/eurecom-s3/symcc/blob/master/docs/Fuzzing.txt): just
prefix the target command with `x86_64-linux-user/symqemu-x86_64`. (Note that
you'll have to run AFL in QEMU mode by adding `-Q` to its command line; the
fuzzing helper will automatically pick up the setting and use QEMU mode too.)

## Build with Docker
First, make sure to have an up-to-date Docker image of SymCC. If you
don't have one, either in the submodule, run:

``` shell
git submodule update --init --recursive symcc
cd symcc
docker build -t symcc .
cd ..
```

Then build the SymQEMU image with (this will also run the tests):
```shell
docker build -t symqemu .
```

You can use the docker with:
```shell
docker run -it --rm symqemu
```

## Contributing

Use the GitHub project for reporting issues, and proposing changes.

### Issues

Please try to provide a minimal test case that demonstrates the problem, or ways
to reproduce the behavior. If possible provide a precise line number if
referring to some code. Ideally, make a PR with the test case demonstrating the
failure (see next point).

### Pull Requests

Pull requests are very welcome. Pull requests will only be merged if all tests
pass, and ideally with a new test case to validate the correctness of the
proposed modifications. QEMU tests that are not specific to SymQEMU should pass
(no regression).

It is very valuable to also make a PR to add a test case for a known bug, this
will facilitate correcting the issue.

Current SymQEMU tests are run by the CI from the Docker container, the following
test suites are currently in place:
- [Unit tests](tests/unit/check-sym-runtime.c): Those tests are made to validate
  specific instrumentation.
- [Integration tests](tests/symqemu/): Those tests are running SymQEMU on a set
  of binaries and compare the results to expected results. Note that those test
  cases can legitimately fail if some changes are made to SymQEMU (because for
  example, an improvement leads to generating new test cases). In that case,
  update the relevant files in `expected_outputs` folders. It would be nice to
  also validate those changes with a new test case.

Also, refer to [QEMU's own tests suite documentation](https://www.qemu.org/docs/master/devel/testing.html).

## Documentation

The [paper](http://www.s3.eurecom.fr/tools/symbolic_execution/symqemu.html)
contains details on how SymQEMU works. A large part of the implementation is the
run-time support in `accel/tcg/tcg-runtime-sym.{c,h}` (which delegates any
actual symbolic computation to SymCC's symbolic backend), and we have modified
most code-generating functions in `tcg/tcg-op.c` to emit calls to the runtime.
For development, configure with `--enable-debug` for run-time assertions; there
are tests for the symbolic run-time support in `tests/check-sym-runtime.c`.

More information about the port to QEMU 8 and internals of (Sym)QEMU
can be found in the QEMU 8 merge commit message
[23e570bf42](https://github.com/eurecom-s3/symqemu/commit/23e570bf42531bcac66a54283eafd4c9c233891a) which
gives some information about the adaptations performed. There are also
some detailed explanations about potential problems in section 5.1 of
Damien Maier's [bachelor
thesis](https://dmaier.ch/bachelor-thesis.pdf).

## License

SymQEMU extends the QEMU emulator and our contributions to previously existing
files adopt those files' respective licenses; the files that we have added are
made available under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 2 of the License, or (at your
option) any later version.
