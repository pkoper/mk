# GNU make utilities (KISS)

Simple solutions for common Makefile challenges.


## chains.mk - chain execution of multiple targets in the desired order

### Problem

```
all: install
install: run_steps_in_order
run_steps_in_order: first second last
first :  ; @echo first
second:  ; @echo second
last  :  ; @echo last
```
The common misconception about the above Makefile code is that executing `make install` will run `first` target, then `second` target and then `last` target.

In parallel execution mode, e.g. `make -j4 install`, targets `first`, `second` and `last` are all run in parallel.

### Manual solution

```
all: install
install: _last
_last: _second
_second: _first
first   _first: ; @echo first
second _second: ; @echo second
last   _last:   ; @echo last
```
In the example above a separate dependency path is created: `install` requires `_last` which requires `_second` which requires `_first`.

The recipie for `_first` ("@echo first") is shared with `first` target, the recipie `_second` is shared with `second` target and so on.

Executing `make install` or `make -j4 install` will first run recipie for `_first`, then recipie for `_second`, then recipie for `_last`.


### Solution using chains.mk

```
all: install

@install = first second last
include chains.mk

install: @install

first   first@install:  ; @echo first
second second@install:  ; @echo second
last     last@install:  ; @echo last
```
Usage:
```
$ make -j4 install
first
second
last
$ make -j4 second
second
$
```

### Multiple execution chains using chains.mk

```
all: install

@install = first second last
@selected = last first
include chains.mk

install: @install
selected: @selected

first   first@install first@selected:  ; @echo first
second second@install               :  ; @echo second
last     last@install  last@selected:  ; @echo last
```
Example:
```
$ make -j4 install
first
second
last
$ make -j4 selected
last
first
$
```

### Rationale for chains.mk

- Simple syntactic sugar for ordered execution (KISS)
- Keeps Makefiles tidy
- Easy target reordering

### Usage

1. Define `@chain1 = step1 step2 step3`
2. `include chains.mk`
3. Define `my_target: @chain1`
4. Insert additonal target name `step1@chain1` for recipie1, `step2@chain1` for recipie2 and so on.

#### Before
```
all: install
install: unpack patch restart
unpack:
    tar -zxf archive.tgz
patch:
    patch < archive.path
restart:
    systemctl restart service1
```
#### After
```
all: install

@install = unpack patch restart
include chains.mk

install: @install

unpack unpack@install:
    tar -zxf archive.tgz
patch patch@install:
    patch < archive.path
restart restart@install:
    systemctl restart service1
```
