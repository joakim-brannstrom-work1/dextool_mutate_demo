# About

This is a demonstration of using [dextool mutate](https://github.com/joakim-brannstrom/dextool/tree/master/plugin/mutate) on open source projects.

The intention is to demonstrate that the tool is usable on a wide range of open
source projects in practice and to serve as example of how to use it to others.
I hope it will help you, as a reader, to get started.

## Warning Note

Because this is a demo the projects/demos will always be somewhat out of sync
with the master versions. Maybe in the future if the demos can be automatically
tested via CI integration it will be possible to always keep them in sync. But
there is no time and budget for that at the moment.

If you find any problem with the demos then please create an issue.

# [GoogleTest](https://github.com/joakim-brannstrom/dextool_mutate_demo/tree/googletest)

Googletest is a nice candidate to show the tool because of the extensive test
suite together with the tools builtin googletest test case parser. The parser
mean that when a mutant is killed it is assigned to one or more test cases.
This tracking thus allow the tool to score the tests, find those that kill zero
mutants, *know* when mutants need to be re-tested because tests have changed
etc.

Unfortunately, as with all mutation testing, it can take a while to run the tool.
Googletest exhibit the classic C++ tendencies of a long compilation time for a
minor change to a header. If you want a quick overview I recommend running with
the flag `--schema-only`.

## Setup Notes

Clone the repo. Create a `build` directory, run cmake:

```sh
mkdir -p build
pushd build
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -Dgtest_build_tests=ON -Dgmock_build_tests=ON ..
popd build
```

Run dextool:

```sh
dextool mutate analyze
dextool mutate test --schema-only
dextool mutate report --style html --section summary --section tc_stat --section tc_killed_no_mutants --section tc_unique --section trend
```

# [secp256k1](https://github.com/joakim-brannstrom/dextool_mutate_demo/tree/secp256k1)

This project is a bit harder than googletest. There are no test case analyzer
which mean the fine grained test case tracking is lost and the tool uses
autotools which lead to a harder time setting it up. But do not dispare, it is
still doable and mutation testing will be useful.

## Setup Notes

First we need a `compile_commands.json`. Thanks to the tool `bear` this is
easy. Install it, hug the bear.

The second thing is that dextools automatic injection of the coverage and
schema runtimes lead to link errors. This is easily solved by, as you can see
in the configuration, tell dextol to not inject the runtime. Instead we link
manually to the precompiled versions. Actually, if you can, always prefer the
precompiled because it reduces the compilation time.

```
export DEXTOOL_INSTALL=where/you/installed/dextool
./autogen.sh
LDFLAGS="-L$DEXTOOL_INSTALL/lib -Wl,--whole-archive -ldextool_coverage_runtime -ldextool_schema_runtime -Wl,--no-whole-archive" ./configure

bear -- make check

dextool mutate analyze
dextool mutate test
dextool mutate report --style html --section summary --section tc_stat --section tc_killed_no_mutants --section tc_unique --section trend
```

## Note

I hope this answers nullc's hackernews comment. When he wrote the following:

[secp256k1 on hackernews regarding mutation testing](https://news.ycombinator.com/item?id=26024915)

```
nullc 3 months ago [–]

I've deployed mutation testing extensively in libsecp256k1 for the past five
years or so, to good ends.

Though it's turned up some testing inadequacies here and there and a
substantial performance improvement (https://twitter.com/pwuille/status/1348835954396516353 ),
I don't believe it's yet caught a bug there, but I do feel a lot more confident
in the tests as a result.

I've also deployed it to a lesser degree in the Bitcoin codebase and turned up
some minor bugs as a result of the tests being improved to pass mutation
testing.

The biggest challenge I've seen for most parties to use mutation testing is
that to begin with you must have 100% branch coverage of the code you might
mutate, and very few ordinary pieces of software reach that level of coverage.

The next issue is that in C/C++ there really aren't any great tools that I'm
aware of-- so every effort needs to be homebrewed.

My process is to have some a harness script that:

1. makes a modification (e.g. a python script that does small search and
   replacements one at a time line by line, or just doing it by hand).
2. attempts to compile the code (if it fails, move on to the next change)
3. Compares the hash of the optimized binary to a collection of already tested
   hashes and moves onto the next if it's already been seen.
4. Runs the tests and if the tests pass save off the diff.
5. Goto 1.

Then I go back trough the diffs and toss ones that are obviously no meaningful
effect, and lob the remaining diffs over to other contributors to figure out if
they're false positives or to improve the tests.
```

No tool existed that could do what he needed. Now, as I have demonstrated here,
there is such a tool.

## [fmtlib](https://github.com/joakim-brannstrom/dextool_mutate_demo/tree/fmtlib)

A C++ template heavy library, or so it seems to me. It uses googletest thus the
full test case tracking work out of the box. The interesting factor here is
that the test cases need to be analyzed because a lot of the mutants are
derived from template instantiations.

## Setup Notes

Clone the repo. Create a `build` directory, run cmake:

```sh
mkdir -p build
pushd build
cmake -Dgmock_build_tests=ON ..
popd build
```

Run dextool:

```sh
dextool mutate analyze
dextool mutate test --schema-only
dextool mutate report --style html --section summary --section tc_stat --section tc_killed_no_mutants --section tc_unique --section trend
```

# [libzmq](https://github.com/joakim-brannstrom/dextool_mutate_demo/tree/fmtlib)

The library in itself do not present any specific problems when it comes to
using the tool. There are some failing tests but those are found and excluded
pretty fast by using the provided `test_cmd` generator. It executes each test one
by one to see which ones pass. This is needed because libzmq by default, it
seems, compile and produce tests that only execute OK on *appropriately
configured* Linux systems.

The `test_cmd` filter:

```sh
dextool_git_repo/tools/dextool_mutate_test_cmd.d --filter-failing --test-cmd-dir build/ --conf .dextool_mutate.toml
```

## Setup Notes

I cannot explain why we need to link both with the library and inject the code
in all tests but ohh well. I am probably misunderstanding cmake in some way.
Buildsystems.... Throw symbols at the problem until the linker stops complaining!

```sh
export DEXTOOL_INSTALL=where/you/installed/dextool
pushd build
cmake .. -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DWITH_PERF_TOOL=OFF -DZMQ_BUILD_TESTS=ON -DENABLE_CPACK=OFF -DCMAKE_EXE_LINKER_FLAGS="-L$DEXTOOL_INSTALL/lib -Wl,--whole-archive -ldextool_coverage_runtime -ldextool_schema_runtime -Wl,--no-whole-archive"
popd
```

# [readerwriterqueue](https://github.com/joakim-brannstrom/dextool_mutate_demo/tree/readerwriterqueue)

A header only library. It is widely used and used by, from what I can see, used
by many. Lets see how good the test suite is. Maybe we will find some googides
here and an opportunity to contribute back.

## Setup Notes

I have modified the cmake build system to make it easier to perform mutation
testing with dextool. The modification mean that the tests are build by cmake
thus a simple `make -j` is all that is needed in the build directory. A side
effect is that cmake generate a complete `compile_commands.json` for us.
Otherwise we would probably need to analyze the tests separately. Because the
library is mostly C++ templates they need to be instantiated for an analysis to
be *complete*.

```sh
mkdir build
pushd build
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
popd
```

# [mosquitto](https://github.com/joakim-brannstrom/dextool_mutate_demo/tree/mosquitto)

Interesting project wherein most tests are written in python. This is all fine
and not a problem when running mutation testing.

The improvements that can be made to the configuration is to:

* either enhance `ptest` to exit on first failed test OR change the
  system/configuration in such a way that dextool can run the tests. Then
  dextool will halt testing on the first failing test.
* add a test case analyzer.

## Setup Notes

Dextool need a compile_commands.json in order to analyze the source code. This
is easiest created by letting cmake do its magic. So the cmake build directory
is only used for anything else than analyzing the source code.

```sh
mkdir build
pushd build
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
popd
```
