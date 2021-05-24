# LIBE2AP

## Table of contents
* [Introduction] (#introduction)
* [Project folders structure](#project-folders-structure)
* [Installation guide](#installation-guide)
  * [Compiling code](#compiling-code)

## Introduction
This repo contains code that is generated from ASN1 descriptions
as well as CMake bits that allow it to be used as a submodule
to other projects.

## Project folder structure

```
├── src
│   ├── e2ap        // E2AP v1.0 message structure
│   └── wrapper     // E2AP message encoding and decoding
```

## Installation guide

### Compiling code

Use cmake and then make to build code from project root.

```
$ cmake .
+++ riclibe2ap library install target directory: lib
+++ mmp version from tag: '1;0;0'
+++ pkg name: riclibe2ap-rotten_1.0.0_amd64.deb
### make package will generate only deb package; cannot find support to generate rpm packages
+++ profiling is off
-- Configuring done
-- Generating done
-- Build files have been written to: <path>/libe2ap
```

Use make to generate shared library (.so).

```
$ sudo make 
```

Use make to install shared library (.so).

```
$ sudo make install
```