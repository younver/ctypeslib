# ctypeslib with libclang


![Build Status](https://github.com/trolldbois/ctypeslib/workflows/ctypeslib-linux/badge.svg)

[![Coverage Status](https://coveralls.io/repos/github/trolldbois/ctypeslib/badge.svg?branch=master)](https://coveralls.io/github/trolldbois/ctypeslib?branch=master)

[![Latest release](https://img.shields.io/github/tag/trolldbois/ctypeslib.svg)]()
[![Supported versions](https://img.shields.io/pypi/pyversions/ctypeslib2.svg)]()

![PyPI](https://img.shields.io/pypi/v/ctypeslib)
![Python](https://img.shields.io/pypi/pyversions/ctypeslib)

[Quick usage guide](docs/ctypeslib_2.0_Introduction.ipynb) in the docs/ folder.

## Status update

 - 2023-04:
   - Please read the installation instructions
 - 2021-02:
   - Thanks for the pull requests
   - Note: libclang-xx-dev must be installed for stddef and other reasons.
   - bump to libclang-11
 - 2018-01-03: master branch works with libclang-5.0 HEAD, python clang from pypi, python3
 - 2017-05-01: master branch works with libclang-4.0 HEAD

## Installation

### LLVM Clang library
First, you should install LLVM clang.
See the LLVM Clang instructions at http://apt.llvm.org/ or use your distribution's packages.

Either use an installer relevant for your OS (APT, downloads, etc..) to install libclang 
```sh
$ sudo apt install libclang1-11
```

or you can use the LLVM install script that installs the whole llvm toolkit 
```sh
 wget https://apt.llvm.org/llvm.sh && chmod +x llvm.sh
 # then install llvm version 16 
 ./llvm.sh 16
 # or version 11 or any other version
 ./llvm.sh 11
```
or you can use anaconda, or any local installation of your favorite choice

### ctypeslib and python packages
Then, install ctypeslib2 and the clang python package with the **same version as your llvm clang library**.

Stable Distribution is available through PyPi at https://pypi.python.org/pypi/ctypeslib2/
if you are not using the latest LLVM clang version, you will need to specify the correct clang python package version

  - If you have installed the latest llvm version: `pip install ctypeslib2` should work fine
  - If you are using llvm clang 16: `pip install ctypeslib2 clang==16`
  - If you are using llvm clang 14: `pip install ctypeslib2 clang==14` 
  - If you are using llvm clang 11: `pip install ctypeslib2 clang==11`
  - etc...

### Alternate configurations

On Ubuntu, libclang libraries are installed with version in the filename.
This library tries to load a few different versions to help you out. (`__init__.py`)
But if you encounter a version compatibility issue, you might have to fix the problem
using one of the following solutions:

1. set the CLANG_LIBRARY_PATH environmental variable to the clang library file or path
```sh
$ export CLANG_LIBRARY_PATH=/lib/x86_64-linux-gnu/libclang-11.so.1
$ clang2py --version
versions - clang2py:2.3.3 clang:11.1.0 python-clang:11.0
```
 
2. OR Install the development package libclang-<version\>-dev to get a file called libclang.so

`$ sudo apt get install libclang-11-dev`
2. OR create a link to libclang-<version\>.so.1 named libclang.so
3. OR hardcode a call to clang.cindex.Config.load_library_file('libclang-<version\>.so.1') in your code before importing ctypeslib


## Usage
### Use ctypeslib2 as a Library in your own python code

```py
import ctypeslib
py_module = ctypeslib.translate('''int i = 12;''')
print(py_module.i)  # Prints 12

py_module2 = ctypeslib.translate('''struct coordinates { int i ; int y; };''')
print(py_module2.struct_coordinates)  # <class 'struct_coordinates'>
print(py_module2.struct_coordinates(1,2))  # <struct_coordinates object at 0xabcde12345>

# input files, output file
py_module3 = ctypeslib.translate_files(['mytest.c'], outfile=open('mytest.py', 'w'))
print(open('mytest.py').read())

# input files, output code
py_module4 = ctypeslib.translate_files(['mytest.c'])
print(open('mytest.py').read())

# input files, output code, with clang options, like cross-platform
from ctypeslib.codegen import config
cfg = config.CodegenConfig()
cfg.clang_opts.extend(['-target', 'arm-gnu-linux'])
py_module5 = ctypeslib.translate_files(['mytest.c'], cfg=cfg)
print(open('mytest.py').read())
```

Look at `test/test_api.py` for more advanced Library usage


### Use ctypeslib2 on the command line

Source file:
```c
// t.c 
struct my_bitfield {
    long a:3;
    long b:4;
    unsigned long long c:3;
    unsigned long long d:3;
    long f:2;
};
```

Run c-to-python script:

    clang2py t.c

Output:
```py
# -*- coding: utf-8 -*-
#
# TARGET arch is: []
# WORD_SIZE is: 8
# POINTER_SIZE is: 8
# LONGDOUBLE_SIZE is: 16
#
import ctypes

class struct_my_bitfield(ctypes.Structure):
    _pack_ = True # source:False
    _fields_ = [
    ('a', ctypes.c_int64, 3),
    ('b', ctypes.c_int64, 4),
    ('c', ctypes.c_int64, 3),
    ('d', ctypes.c_int64, 3),
    ('f', ctypes.c_int64, 2),
    ('PADDING_0', ctypes.c_int64, 49)]

__all__ = \
    ['struct_my_bitfield']
```

### use ctypeslib with additional clang arguments:

Source file:
```c
// test-stdbool.c 
#include <stdbool.h>

typedef struct s_foo {
    bool bar1;
    bool bar2;
    bool bar3;
} foo;
```

Run c-to-python script (with any relevant include folder):

    clang2py --clang-args="-I/usr/include/clang/4.0/include" test-stdbool.c

Output:
```py
# -*- coding: utf-8 -*-
#
# TARGET arch is: ['-I/usr/include/clang/4.0/include']
# WORD_SIZE is: 8
# POINTER_SIZE is: 8
# LONGDOUBLE_SIZE is: 16
#
import ctypes

class struct_s_foo(ctypes.Structure):
    _pack_ = True # source:False
    _fields_ = [
    ('bar1', ctypes.c_bool),
    ('bar2', ctypes.c_bool),
    ('bar3', ctypes.c_bool),]

foo = struct_s_foo
__all__ = ['struct_s_foo', 'foo']
```

## _pack_ and PADDING explanation

    clang2py test/data/test-record.c

This outputs:
```py
# ...

class struct_Node2(Structure):
    _pack_ = True # source:False
    _fields_ = [ 
    ('m1', ctypes.c_ubyte),
    ('PADDING_0', ctypes.c_ubyte * 7),
    ('m2', POINTER_T(struct_Node)),]

# ...
```
The PADDING_0 field is added to force the ctypes memory Structure to align fields offset with the definition given
by the clang compiler.

The [_pack_](https://docs.python.org/3/library/ctypes.html#ctypes.Structure._pack_) attribute forces the alignment 
on 0 bytes, to ensure all fields are as defined by this library, and not per the compiler used by the host python binary

The objective of this, is to be able to produce cross-architecture python code, that can read memory structures from a 
different architecture (like reading a memory dump from a different architecture)

See `clang-11 -print-targets` for options


## Usage details

    usage: clang2py [-h] [-c] [-d] [--debug] [-e] [-k TYPEKIND] [-i] [-l DLL] [-m module] [--nm NM] [-o OUTPUT] [-p DLL] [-q] [-r EXPRESSION] [-s SYMBOL] [-t TARGET] [-v] [-V] [-w W] [-x] [--show-ids SHOWIDS] [--max-depth N]
                    [--validate VALIDATE] [--clang-args CLANG_ARGS]
                    files [files ...]
    
    Version 2.3.3. Generate python code from C headers
    
    positional arguments:
      files                 source filenames. stdin is not supported
    
    options:
      -h, --help            show this help message and exit
      -c, --comments        include source doxygen-style comments
      -d, --doc             include docstrings containing C prototype and source file location
      --debug               setLevel to DEBUG
      -e, --show-definition-location
                            include source file location in comments
      -k TYPEKIND, --kind TYPEKIND
                            kind of type descriptions to include: a = Alias, c = Class, d = Variable, e = Enumeration, f = Function, m = Macro, #define s = Structure, t = Typedef, u = Union default = 'cdefstu'
      -i, --includes        include declaration defined outside of the sourcefiles
      -l DLL, --include-library DLL
                            library to search for exported functions. Add multiple times if required
      -m module, --module module
                            Python module(s) containing symbols which will be imported instead of generated
      --nm NM               nm program to use to extract symbols from libraries
      -o OUTPUT, --output OUTPUT
                            output filename (if not specified, standard output will be used)
      -p DLL, --preload DLL
                            dll to be loaded before all others (to resolve symbols)
      -q, --quiet           Shut down warnings and below
      -r EXPRESSION, --regex EXPRESSION
                            regular expression for symbols to include (if neither symbols nor expressions are specified,everything will be included)
      -s SYMBOL, --symbol SYMBOL
                            symbol to include (if neither symbols nor expressions are specified,everything will be included)
      -t TARGET, --target TARGET
                            target architecture (default: x86_64-Linux)
      -v, --verbose         verbose output
      -V, --version         show program's version number and exit
      -w W                  add all standard windows dlls to the searched dlls list
      -x, --exclude-includes
                            Parse object in sources files only. Ignore includes
      --show-ids SHOWIDS    Don't compute cursor IDs (very slow)
      --max-depth N         Limit cursor expansion to depth N
      --validate VALIDATE   validate the python code is correct
      --clang-args CLANG_ARGS
                            clang options, in quotes: --clang-args="-std=c99 -Wall"
    
    Cross-architecture: You can pass target modifiers to clang. For example, try --clang-args="-target x86_64" or "-target i386-linux" to change the target CPU arch.


## Inner workings for memo

- clang2py is a script that calls ctypeslib/ctypeslib/clang2py.py
- clang2py.py is mostly the old xml2py.py module forked to use libclang.
- clang2py.py calls ctypeslib/ctypeslib/codegen/codegenerator.py
- codegenerator.py calls ctypeslib/ctypeslib/codegen/clangparser.py
- clangparser.py uses libclang's python binding to access the clang internal representation of the C source code. 
    - It then translate each child of the AST tree to python objects as listed in typedesc.
- codegenerator.py then uses these python object to generate ctypes-based python source code.
 
Because clang is capable to handle different target architecture, this fork 
 {is/should be} able to produce cross-platform memory representation if needed.


## Credits

This fork of ctypeslib is mainly about using the libclang1>=3.7 python bindings
to generate python code from C source code, instead of gccxml.

the original ctypeslib contains these packages:
 - ``ctypeslib.codegen``       - a code generator
 - ``ctypeslib.contrib``       - various contributed modules
 - ``ctypeslib.util``          - assorted small helper functions
 - ``ctypeslib.test``          - unittests

This fork of ctypeslib is heavily patched for clang.
- https://github.com/trolldbois/ctypeslib is based on rev77594 of the original ctypeslib.
- git-svn-id: http://svn.python.org/projects/ctypes/trunk/ctypeslib@775946015fed2-1504-0410-9fe1-9d1591cc4771

The original ctypeslib is written by
- author="Thomas Heller",
- author_email="theller@ctypes.org",
