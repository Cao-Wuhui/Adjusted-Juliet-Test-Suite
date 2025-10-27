# Juliet Test Suite for C/C++

This is the Juliet Test Suite for C/C++ version 1.3 from https://samate.nist.gov/SARD/testsuite.php augmented with a build system for Unix-like OSes that supports automatically building test cases into individual executables and running those tests. The build system originally provided with the test suite supports building all test cases for a particular [CWE](https://cwe.mitre.org/) into a monolithic executable. Building individual test cases supports the evaluation of projects like ASan that facilitate memory safety for C/C++ programs at runtime. 

Testcases are organized by CWE in the `testcases` subdirectory. `juliet.py` is the main script that supports building and running individual test cases - individual CWEs or the entire test suite can be targeted. To build executables, `juliet.py` copies `CMakeLists.txt` into the directories for targeted CWEs and runs cmake followed by make. Output appears by default in a `bin` subdirectory. Each targeted CWE has a `bin/CWEXXX` directory that is further divided into `bin/CWEXXX/good` and `bin/CWEXXX/bad` subdirectories. For each test case, a "good" binary that does not contain the error is built and placed into the good subdirectory and a "bad" binary that contains the error is built and placed into the bad subdirectory.

To run the executables after they are built, `juliet.py` invokes the `juliet-run.sh` script, which is copied to the `bin` subdirectory during the build. It records exit codes in `bin/CWEXXX/good.run` and `bin/CWEXXX/bad.run`. Executables are run with a timeout so that test cases requiring user input timeout with exit code 124.

**Note:** Some Juliet C++ test cases that use `namespace std` and the `bind()` socket function fail to compile under C++11 due to the introduction of `std::bind()`. This version replaces `bind()` calls in C++ source files with `::bind()`.

## Motivation

This repository adjusts the Juliet Test Suite to better support runtime evaluation of memory safety tools (e.g., ASan and Tech-ASan) with reproducible results. In particular, we:
- build per-testcase executables and provide a non-interactive runner;
- improve reproducibility by removing randomness and filtering non-deterministic or platform-dependent tests;
- unify inputs/timeouts and fix C++ `bind()` conflicts under C++11;
- provide ASan-friendly build settings and a statistics script compatible with older Python 3.

If this repository or the scripts are helpful to your research, please cite:

```bibtex
@inproceedings{cao2025tech-asan,
  title     = {Tech-ASan: Two-stage check for Address Sanitizer},
  author    = {Cao, Yixuan and Feng, Yuhong and Li, Huafeng and Huang, Chongyi and Jian, Fangcao and Li, Haoran and Wang, Xu},
  booktitle = {Proceedings of the 16th International Conference on Internetware},
  year      = {2025},
  doi       = {10.1145/3755881.3755918}
}
```

## Running Sample

Clean, build, and run tests.

``` shell
python3 juliet.py 121 122 124 126 127 -o ./bin -c -g -m -r
```

Show statistical test results.

``` shell
python3 parse-cwe-status.py ./bin/CWE126/bad.run
```

An example of statistical results is shown below.

``` shell
===== EXIT STATUS =====
OK            25
1            647

===== DATAFLOW VARIANTS =====
 VAR         OK         1
  1:          1        14
  2:          1        14
...

===== FUNCTIONAL VARIANTS =====
                                      OK         1
CWE129_fgets                           0        48
CWE129_fscanf                          0        48
CWE129_large                          25        23
char_alloca_loop                       0        40
char_alloca_memcpy                     0        40
char_alloca_memmove                    0        40
...
```

## Modify dataset

To make results reproducible across runs, we remove the effect of randomness from `rand()` by changing the return value of `globalReturnsTrueOrFalse()` to `1` (JTS flow variant 12 calls this function).

``` C
int globalReturnsTrueOrFalse() 
{
    // return (rand() % 2);
    return 1;
}
```

## Compiler settings

Set the compiler and enable AddressSanitizer compilation options.

``` bash
# Set the C and C++ compilers
set(CMAKE_C_COMPILER "/usr/bin/clang-4.0")
set(CMAKE_CXX_COMPILER "/usr/bin/clang++-4.0")

project("juliet-c-${CWE_NAME}")

# Set the C and C++ compiler flags
set(CMAKE_C_FLAGS "-fsanitize=address -fsanitize-recover=address")
set(CMAKE_CXX_FLAGS "-fsanitize=address -fsanitize-recover=address")
```

## Filter dataset

To further improve reproducibility, we adjust the following tests:
1. Tests whose names contain `socket` require both client and server. They are not suitable for AddressSanitizer tests and often cause non-deterministic timeouts, leading to inconsistent results.
2. Tests whose names contain `rand` may also yield inconsistent results due to randomness.
3. Tests whose names contain `CWE170` print a string without the terminating character `\0`, which can lead to unpredictable overflows.
4. Tests whose names contain `sizeof_` trigger errors on 32-bit systems but not on 64-bit systems.

``` bash
if echo "$TESTCASE" | grep -q "socket"
then continue
fi

if echo "$TESTCASE" | grep -q "rand"
then continue
fi

if echo "$TESTCASE" | grep -q "CWE170"
then continue
fi

if echo "$TESTCASE" | grep -q "sizeof_"
then continue
fi
```

## Process the datasets that require input

Following the [FloatZone test script](https://github.com/vusec/instrumentation-infra/blob/5bfbf68e0cfe46cf9600a0bcf4fa7a4a2fd80e48/infra/targets/juliet.py), we provide input via files. For general input, we pass `11`. For underflow read/write (CWE124, CWE127), we pass `-1`.

``` bash
if [ "$CWDID" = "124" ] || [ "$CWDID" = "127" ]
then
    timeout "${TIMEOUT}" "${TESTCASE_PATH}" < "${INPUT_FILE_124_127}"
else
    timeout "${TIMEOUT}" "${TESTCASE_PATH}" < "${INPUT_FILE}"
fi
```

## Improve the compatibility of the statistics script

The statistical script uses some data structures of higher versions of python3, which are not supported by lower versions of python3. Additional packages need to be imported and the original data structures replaced.

``` python
from typing import List, Dict, Tuple
...
def do_parsing(filename: str) -> Tuple[str, Dict[int, List[int]], Dict[str, Dict[int, int]]]:
    ...
```
