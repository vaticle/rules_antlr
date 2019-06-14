[![Build Status](https://travis-ci.org/marcohu/rules_antlr.png?branch=master)](https://travis-ci.org/marcohu/rules_antlr)
[![Java 8+](https://img.shields.io/badge/java-8+-4c7e9f.svg)](http://java.oracle.com)
[![License](https://img.shields.io/badge/license-Apache2-blue.svg)](https://github.com/marcohu/rules_antlr/blob/master/LICENSE)


# ANTLR Rules for Bazel

These build rules are used for processing [ANTLR](https://www.antlr.org)
grammars with [Bazel](https://bazel.build/). Currently only Java targets are
really usable due to shortcomings with the Bazel rules for other languages.

<a name="toc"></a>


<a name="setup"></a>
## Quick Setup

Add the following to your [`WORKSPACE`](https://docs.bazel.build/versions/master/build-ref.html#workspace)
file to include the external repository and load the necessary dependencies for the [`antlr`](docs/antlr4.md) rule:

```python
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_jar")
...
http_archive(
    name = "rules_antlr",
    sha256 = "932f0637acc20b67d90e68e47d019961105b00c5991a72ffee33bc1e58541734",
    strip_prefix = "rules_antlr-0.2.0",
    urls = ["https://github.com/marcohu/rules_antlr/archive/0.2.0.tar.gz"],
)

load("@rules_antlr//antlr:deps.bzl", "antlr_dependencies")
antlr_dependencies()
```

More detailed instructions can be found in the
[Setup](https://github.com/marcohu/rules_antlr/tree/master/docs/setup.md) section.


### Build Rules

To add ANTLR code generation to your [BUILD](https://docs.bazel.build/versions/master/build-ref.html#BUILD_files) files, you first have to load the extension for the desired ANTLR release.

For ANTLR 4:

```python
load("@rules_antlr//antlr:antlr4.bzl", "antlr")
```

For ANTLR 3:

```python
load("@rules_antlr//antlr:antlr3.bzl", "antlr")
```

For ANTLR 2:

```python
load("@rules_antlr//antlr:antlr2.bzl", "antlr")
```

You can then invoke the rule:

```python
antlr(
    name = "parser",
    srcs = ["Hello.g4"],
    package = "hello.world",
)
```

It's also possible to use different ANTLR versions in the same file via aliasing:

```python
load("@rules_antlr//antlr:antlr4.bzl", antlr4 = "antlr")
load("@rules_antlr//antlr:antlr3.bzl", antlr3 = "antlr")

antlr4(
    name = "parser",
    srcs = ["Hello.g4"],
    package = "hello.world",
)

antlr3(
    name = "old_parser",
    srcs = ["OldHello.g"],
    package = "hello.world",
)
```

Refer to the rule reference documentation for the available rules and attributes:

* <a href="#antlr4">ANTLR 4</a>
* <a href="#antlr3">ANTLR 3</a>
* <a href="#antlr2">ANTLR 2</a>


<a name="basic-example"></a>
## Basic Java Example

Suppose you have the following directory structure for a simple ANTLR project:

```
[workspace]/
  WORKSPACE
  HelloWorld/
    BUILD
    src/
      main/
        antlr4/
          Hello.g4
```

`HelloWorld/src/main/antlr4/Hello.g4`

```
grammar Hello;
r  : 'hello' ID;
ID : [a-z]+;
WS : [ \t\r\n]+ -> skip;
```

`HelloWorld/BUILD`

```python
load("@rules_antlr//antlr:antlr4.bzl", "antlr")

antlr(
    name = "generated",
    srcs = ["src/main/antlr4/Hello.g4"],
    package = "hello.world",
)

java_library(
    name = "HelloWorld",
    srcs = [":generated"],
)
```

Compiling the project generates the lexer/parser files:

```
$ bazel build //HelloWorld
INFO: Analysed target //HelloWorld:HelloWorld (0 packages loaded).
INFO: Found 1 target...
Target //HelloWorld:HelloWorld up-to-date:
  bazel-bin/HelloWorld/libHelloWorld.jar
INFO: Elapsed time: 0.940s, Critical Path: 0.76s
INFO: Build completed successfully, 4 total actions
```

The generated source files can be found in the `generated.srcjar` archive below your workspace `bazel-bin/HelloWorld` directory.

To just generate the source files you would use:

    $ bazel build //HelloWorld:generated

Refer to the [examples](examples)
directory for further samples.


## Project Layout

ANTLR rules will store all generated source files in a `target-name.srcjar` zip archive below your workspace [`bazel-bin`](https://docs.bazel.build/versions/master/output_directories.html#documentation-of-the-current-bazel-output-directory-layout) folder.
Depending on the ANTLR version, there are three ways to control namespacing and directory structure for generated code, all with their pros and cons.

1. The [`package`](#antlr4.package) rule attribute ([`antlr4`](#antlr4) only). Setting the namespace via the [`package`](#antlr4.package) attribute will generate the corresponding target language specific namespacing code (where applicable) and puts the generated source files below a corresponding directory structure. To not create the directory structure, set the [`layout`](#antlr4.layout) attribute to `flat`.<br>Very expressive and allows language independent grammars, but only available with ANTLR 4, requires several runs for different namespaces, might complicate refactoring and can conflict with language specific code in `@header {...}` sections as they are mutually exclusive.

2. Language specific application code in grammar `@header {...}` section. To not create the corresponding directory structure, set the [`layout`](#antlr4.layout) attribute to `flat`.<br>Allows different namespaces to be processed in a single run and will not require changes to build files upon refactoring, but ties grammars to a specific language and can conflict with the [`package`](#antlr4.package) attribute as they are mutually exclusive.

3. The project layout ([`antlr4`](#antlr4) only). Putting your grammars below a common project directory will determine namespace and corresponding directory structure for the generated source files from the relative project path. ANTLR rules uses different defaults for the different target languages (see below), but you can define the root directory yourself via the [`layout`](#antlr4.layout) attribute.<br>Allows different namespaces to be processed in a single run without language coupling, but requires conformity to a specific (albeit configurable) project layout and the [`layout`](#antlr4.layout) attribute for certain languages.


### Common Project Directories

The [`antlr4`](#antlr4) rule supports a common directory layout to figure out namespacing from the relative directory structure. The table below lists the default paths for the different target languages. The version number at the end is optional.

| Language                 | Default Directory<span style="display:inline-block;width:4em"/>|
|--------------------------|------------------|
| C                        | `src/antlr4`     |
| Cpp                      | `src/antlr4`     |
| CSharp, CSharp2, CSharp3 | `src/antlr4`     |
| Go                       | &nbsp;           |
| Java                     | `src/main/antlr4`|
| JavaScript               | `src/antlr4`     |
| Python, Python2, Python3 | `src/antlr4`     |
| Swift                    |  &nbsp;          |

For languages with no default, you have to set your preference with the [`layout`](#antlr4.layout) attribute.
