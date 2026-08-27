# llvm

Thin Carp bindings over the [LLVM-C API](https://llvm.org/doxygen/group__LLVMC.html):
build IR, verify it, print it, and JIT-compile it to native code you can call
back into from Carp.

Two layers, following the `lua`/`luax` precedent:

- `LLVM` — low-level 1:1 `register` bindings over the C API (raw handles).
- `LLVMx` — safe conveniences (`with-context`, `with-builder`).

## Prerequisites

LLVM must be installed with its development headers and a linkable `libLLVM`.
On macOS via Homebrew:

```
brew install llvm
```

`LLVM.setup` defaults to the Homebrew keg at `/opt/homebrew/opt/llvm`. Pass a
different prefix (e.g. `llvm-config --prefix`) if yours lives elsewhere:

```
(LLVM.setup "/usr/lib/llvm-18")
```

## Usage

```clojure
(load "git@github.com:carpentry-org/llvm@0.1.0")
(LLVM.setup)

(use LLVM)

; i32 add(i32 a, i32 b) { return a + b; }
(defn build []
  (let-do [ctx (context-create)
           builder (builder-create ctx)
           mod (module-create (String.cstr &@"demo") ctx)
           params [(int32-type ctx) (int32-type ctx)]
           fnty (function-type (int32-type ctx) &params false)
           func (add-function mod (String.cstr &@"add") fnty)
           entry (append-block ctx func (String.cstr &@"entry"))]
    (position-at-end builder entry)
    (build-ret builder
      (build-add builder (get-param func 0) (get-param func 1)
                 (String.cstr &@"sum")))
    (IO.println &(print-module mod))       ; textual IR
    (IO.println &(verify-module mod))      ; "" if valid

    ; JIT it and run add(20, 22)
    (link-in-mcjit)
    (ignore (initialize-native-target))
    (ignore (initialize-native-asm-printer))
    (let-do [engine (create-execution-engine mod)]
      (let [addr (function-address engine (String.cstr &@"add"))]
        (IO.println &(fmt "add(20, 22) = %d" (call-i32-i32-i32 addr 20 22))))
      (engine-dispose engine))
    (builder-dispose builder)
    (context-dispose ctx)))
```

## Ownership

The opaque handle types are declared at top level (`LLVMContext`, `LLVMModule`,
`LLVMType`, `LLVMValue`, `LLVMBasicBlock`, `LLVMBuilder`, `LLVMEngine`) and bound
by value, like cairo's handles.

Only the root resources — `LLVMContext`, `LLVMModule`, `LLVMBuilder`,
`LLVMEngine` — are ever disposed. `LLVMValue`, `LLVMType`, and `LLVMBasicBlock`
are owned by their context/module and must never be freed individually, so they
carry no `delete`. An execution engine takes ownership of the module handed to
it, so `engine-dispose` frees that module too — do not also dispose it yourself.

## Scope

A broad working subset (~139 LLVM-C functions across 10 headers, 53 of the 112
`LLVMBuild*` instructions), enough to lower and ship a real language frontend:

- **Types** — int (1/8/16/32/64), float, double, void, pointer, array, function,
  vector, and struct types (anonymous and named/recursive via
  `struct-create-named` + `struct-set-body`).
- **Constants** — integer, float, null, and constant vectors.
- **Instructions** — full integer/bitwise/float arithmetic, all the casts
  (trunc/ext/fp↔int/ptr↔int/bitcast), integer and float comparison, `select`,
  `getelementptr` (incl. `struct-gep`), aggregate and vector extract/insert,
  `shufflevector`, `alloca`, `load`, `store`, branches, `phi`, and `switch`.
- **Module structure** — functions, params, basic blocks, global variables
  (initializers, constness, linkage), interned global strings, named lookups,
  attributes (enum + string), metadata (nodes, named metadata, module flags,
  instruction metadata).
- **Debug info** — DWARF via a `DIBuilder`: files, compile units, basic and
  subroutine types, subprograms, and source locations.
- **Backends** — verification, optimization (`run-passes` with the new pass
  manager, e.g. `"default<O2>"`), MCJIT execution, AOT emission of a native
  object/assembly file, bitcode writing, and reading textual IR or bitcode back
  in (`memory-buffer-of-file` + `parse-ir`).

It targets the callback-light MCJIT path rather than ORC (capturing Carp
closures do not become C function pointers), and binds the host target for
codegen. Not covered: the ORC/LLJIT APIs, the full `DIBuilder` surface (local
variables, composite/derived types, expressions), attribute introspection, and
the linker.

## Example: a tiny language

`examples/mini.carp` is a complete little compiler built on these bindings — a
Lisp-ish expression language with integers, arithmetic, `<`, `if`, and named
recursive functions:

```
(defn fib (n) (if (< n 2) n (+ (fib (- n 1)) (fib (- n 2)))))
(defn gcd (a b) (if (< b 1) a (gcd b (- a (* (/ a b) b)))))
```

It lexes and parses source to an AST, lowers each function to LLVM IR (`if`
becomes blocks + a `phi`, `<` an `icmp`+`zext`, calls resolve by name so
recursion just works), verifies the module, JIT-executes it, and emits a native
`out/mini.o`:

```
carp -x examples/mini.carp
# ... prints the generated IR, then:
# fib(10) = 55   fib(20) = 6765   gcd(48, 36) = 12   gcd(1071, 462) = 21
```

The emitted object links like any other: `cc your_main.c out/mini.o` produces a
working binary that calls `fib`/`gcd` directly.

## Tests

```
carp -x test/llvm.carp       # build → verify → JIT an add function
carp -x test/frontend.carp   # structs, casts, GEP, switch, globals; JIT + emit a .o
carp -x test/advanced.carp   # passes, vectors, attributes, debug info, IR/bitcode round-trip
```

`frontend.carp` lowers two functions and emits a real host object file to
`out/frontend.o` — inspect it with `nm out/frontend.o` or `otool -tvV`.
`advanced.carp` runs `default<O2>` (and checks a dead alloca is gone), builds a
`<4 x i32>` vector op, attaches `noinline` and a `DISubprogram`, and writes then
re-parses `out/io.ll` / `out/io.bc`.
