---
title: Makefile Notes
date: 2021-06-22 21:20:15
summary:
tags:
  - make
categories: technical
---

My notes for the `Makefile` file used by the `make` utility.

## Assignments
The simplest `=` will expand recursively when the LHS is used
```
a = "1"
b = "2"
c = $(a)3 # c is now "13"
a = "4"   # c is now "43"
```

----

`:=` will expand and lock-in the resulting value when it assigns
```
a = "1"
b = "2"
c := $(a)3 # c will remain "13"
a = "4"    # c is still "13"
```

----

`?=` will assign only if the LHS is not assigned to anything
```
a = "1"
a ?= "2" # a is still "1"
```

----

## Magic variables

* `$@`: target
* `$<`: the first prerequisite
* `$^`: list of all prerequisite
* `$*`: stem with which the implicit rule matches ("foo" in "foo.c")
* `$(@D)`: target directory

## Basic implicit rule
```
%.o: %.c
    command
```
means every `*.o` file seen in the make process is added a dependency `*.c`, recipe is given in `command`

## List generation
```
C_FILES = $(wildcard $(SRC_DIR)/*.c)
```
produces a list `C_FILES` which contains all the `*.c` files in `$(SRC_DIR)`

## List mapping

```
OBJ_FILES = $(C_FILES:$(SRC_DIR)/%.c=$(BUILD_DIR)/%_c.o)
```
`OBJ_FILES` is produced by mapping every item in `C_FILES` from `$(SRC_DIR/%.c` to `$(BUILD_DIR)/%_c.o`

## Including files
Simply use `include`
```
include myrule.file
```
or use `-include` to not do anything if the file doesn't exist.
```
-include myrule.file # no error if myrule.file doesn't exist
```

## Some `-M` options in gcc
* `-M`: instead of outputing the preprocessed result, output a `make` suiting dependency rule
* `-MM`: ignore system header files
* `-MD`: don't stop at the preprocessor, do the rest of the compilation process and generate the dependency as a side product
* `-MMD`: `-MD` and `-MM` combined
* `-MP`: create phony targets for each dependency other than the main file

### `-MP` explanation
Say the `-MMD` option generated `foo.d`:
```
foo.o: foo.c foo.h myheader.h
```
Now if `myheader.h` is no longer needed, we remove it from `foo.c`, delete `myheader.h` and rerun `make`, because `foo.d` hasn't changed `make` will complain that `myheader.h` is not found and report an error.
`-MP` will create `foo.d` like this:
```
foo.o: foo.c foo.h myheader.h

foo.h:

myheader.h:
```
Then `myheader.h` will be seen as an empty target by `make` in the case that `myheader.h` is not present.

## Multi-rule handling
When using implicit rules and `include`s, there might be multiple rules for one target, in that case all the dependencies are unioned into one big list. The recipe is executed when any of the dependency is younger than the target. There can only be one recipe to be executed for a file.

## References
[make manual: 4.11 Multiple Rules for One Target](https://www.gnu.org/software/make/manual/html_node/Multiple-Rules.html#Multiple-Rules)

[makefile文件中dash include的含义](https://blog.csdn.net/howellzhu/article/details/43083675)

[Makefile cheatsheet](https://devhints.io/makefile)

[Your Makefiles are Wrong](https://tech.davis-hansson.com/p/make/)
