---
layout: ../layouts/PageLayout.astro
title: 'Gardel Language Guide'
description: 'An evolving overview of Gardel’s core concepts, syntax, and verification model.'
status: 'Living Draft'
---

> **Living Guide:** Gardel is actively evolving. Each section introduces a core concept concisely so the guide can grow and change alongside the language.

## Contents

1. [Philosophy & Roots](#1-philosophy--roots)
2. [Primitives & Safe Numerics](#2-primitives--safe-numerics)
3. [Records, Sum Types & Proof-Aware Matching](#3-records-sum-types--proof-aware-matching)
4. [Contracts & Hoare Triples](#4-contracts--hoare-triples)
5. [Modules, Visibility & Uniform Access](#5-modules-visibility--uniform-access)
6. [Total Recursion & Termination](#6-total-recursion--termination)
7. [Higher-Order Functions & Refined Types](#7-higher-order-functions--refined-types)
8. [Concepts & Algebraic Laws](#8-concepts--algebraic-laws)
9. [Deterministic Parallelism](#9-deterministic-parallelism)
10. [Effects: `program` & `reactive program`](#10-effects-program--reactive-program)
11. [Compiler CLI Quick Reference](#11-compiler-cli-quick-reference)

---

## 1. Philosophy & Roots

Gardel is a **pure, total, statically verified** programming language designed to move correctness guarantees—null safety, bounds safety, overflow safety, termination, and functional specifications—into compile-time proofs.

Its design draws on four foundations:

- **Tony Hoare's Axiomatic Semantics:** Every routine specifies a Hoare triple (`require` / body / `ensure`) checked statically by Boogie and Z3.
- ***Object-Oriented Software Construction* (2nd ed.) / Eiffel:** Contract syntax, `Current` and `Result`, class invariants, Uniform Access, and selective export visibility.
- **Functional Programming (Bird & Wadler):** Pure value transformers, algebraic data types, structural recursion, and equational reasoning from *Introduction to Functional Programming*.
- **Bend:** Lock-free, deterministic parallelism derived from pure divide-and-conquer recursion.

---

## 2. Primitives & Safe Numerics

Runtime values use native 64-bit machine types (`NAT`, `U64`, `I64`, `F64`) alongside `BOOL`, `CHAR`, and `STRING`, while contracts reason over unbounded mathematical integers:

- Standard `+`, `-`, `*`, and `/` are **checked**: the verifier must prove no overflow or division by zero can occur.
- Wrapping machine arithmetic uses explicit `%` variants (`+%`, `-%`, `*%`) when wrap-around is intended.

```gardel
midpoint(lo: NAT, hi: NAT): NAT
  require
    ordered: lo <= hi
  do
    lo + (hi - lo) / 2   -- proved not to overflow
  ensure
    in_range: lo <= Result and Result <= hi
  end
```

---

## 3. Records, Sum Types & Proof-Aware Matching

Gardel has no `null` and no in-place mutation. Local `let` bindings are immutable, and data is modeled with algebraic product and sum types.

### Product Types (Records) & Functional Updates

```gardel
module POINT

representation {ANY}
  x: I64
  y: I64

create {ANY}
  origin: POINT
    do
      { x = 0, y = 0 }
    end

feature {ANY}
  translate_x(dx: I64): POINT
    do
      { Current with x = Current.x + dx }
    end

end
```

### Sum Types & Refinement-Aware `match`

Sum types use `|` variants. Pattern matching is **proof-aware**: all reachable cases must be handled, and branches proven unreachable by a precondition must be omitted:

```gardel
module OPTION[A]

representation {ANY}
  | None
  | Some(value: A)

feature {ANY}
  is_some: BOOL
    do
      match Current
        case None    then False
        case Some(_) then True
      end
    end

  get: A
    require
      present: Current.is_some
    do
      -- `case None` is omitted because `present` proves it unreachable
      match Current
        case Some(v) then v
      end
    end

end
```

---

## 4. Contracts & Hoare Triples

Contracts are **static proofs, not runtime assertions**. Once discharged by Boogie and Z3, they are erased from compiled binaries:

- `require` — precondition the caller must establish.
- `ensure` — postcondition guaranteed on return (using `Result`).
- `invariant` — refinement predicate enforced at every construction site of a module.
- `check ... end` — intermediate proof step (for example, invoking an inductive lemma).

```gardel
module BOUNDED_STACK[A]

representation {NONE}
  items: LIST[A]
  size: NAT
  limit: NAT

create {ANY}
  make(cap: NAT): BOUNDED_STACK[A]
    require
      positive_capacity: cap > 0
    do
      { items = LIST.empty, size = 0, limit = cap }
    ensure
      is_empty: Result.count = 0
      capacity_set: Result.capacity = cap
    end

feature {ANY}
  count: NAT
    do
      Current.size
    end

  capacity: NAT
    do
      Current.limit
    end

  push(x: A): BOUNDED_STACK[A]
    require
      not_full: Current.count < Current.capacity
    do
      { items = Current.items.prepend(x),
        size = Current.size + 1,
        limit = Current.limit }
    ensure
      count_incremented: Result.count = Current.count + 1
      capacity_unchanged: Result.capacity = Current.capacity
    end

invariant
  positive_capacity: limit > 0
  size_matches: size = items.length
  within_bounds: size <= limit

end
```

---

## 5. Modules, Visibility & Uniform Access

Each `module` acts as an Abstract Data Type with explicit sections:

| Section | Purpose |
| :--- | :--- |
| `representation {VIS}` | Internal record fields or algebraic sum variants |
| `create {VIS}` | Constructors that establish the module `invariant` |
| `feature {VIS}` | Queries and transformations (`Current` is the receiver) |
| `specify {CONCEPT}` | Proofs that the module satisfies a concept's algebraic laws |

Visibility blocks—such as `{NONE}`, `{ANY}`, or `{FRIENDS}`—control who can inspect representation fields or call routines. By the **Uniform Access Principle**, field reads and zero-argument queries share the same `obj.count` syntax.

---

## 6. Total Recursion & Termination

Pure modules are **total**: every recursive routine must prove termination with a `variant` expression that strictly decreases on every recursive call and stays bounded below (for `NAT` or `INT >= 0` metrics):

```gardel
all_ge(lo: K, xs: LIST[K]): BOOL
  variant
    xs.length
  do
    if xs.length = 0 then
      True
    else
      (not xs.head.less_than(lo)) and BINARY_SEARCH.all_ge(lo, xs.tail)
    end
  end
```

Arithmetic metrics such as `hi - lo` (for binary search) and lexicographic tuple variants are also supported.

---

## 7. Higher-Order Functions & Refined Types

Functions are first-class values (`A -> B`) created with `fn` lambdas or named routines, and compiled via **compile-time lambda lifting** (converting captured variables into immutable records with no GC required).

Function parameters and return types can even carry contracts:

```gardel
apply_small(
  f: (x: NAT) -> NAT require small: x < 100 ensure bounded: Result <= x,
  k: NAT
): NAT
  require
    k_small: k < 100
  do
    f(k)
  ensure
    le: Result <= k
  end
```

---

## 8. Concepts & Algebraic Laws

Instead of implementation inheritance, Gardel uses **concepts**—interfaces paired with verifiable `law` predicates—and proves conformance inside `specify`:

```gardel
concept MONOID
  empty: Current
  combine(other: Current): Current

  law associativity(a: Current, b: Current, c: Current)
    ensure
      assoc: a.combine(b.combine(c)) = a.combine(b).combine(c)
    end

  law left_identity(a: Current)
    ensure
      left_id: empty.combine(a) = a
    end
end
```

Standard concepts in the prelude include `EQ`, `CANONICAL_EQ`, and `ORD`, which constrain generic modules via bounds like `module BINARY_SEARCH[K: ORD]`.

---

## 9. Deterministic Parallelism

Because pure functions have no side effects or shared mutable state, evaluation order cannot change results (Church-Rosser confluence). Independent divide-and-conquer branches—such as balanced `SEQ[A]` reductions—can execute in parallel via C11/OpenMP without locks or data races, with Bend v2 as an evolving parallel target.

---

## 10. Effects: `program` & `reactive program`

Pure `module`s forbid side effects completely. Effects are isolated behind a **Purity Firewall** in two dedicated module kinds:

### Linear `IO[A]` in `program`

An `IO[A]` token must be consumed linearly inside a `with IO ... end` block, using `<-` to bind results and `RESULT` / `OPTION` to ingest untrusted external input safely:

```gardel
program ECHO

feature {ANY}

  main(args: SEQ[STRING]): IO[EXIT_CODE]
    do
      with IO
        line_res <- IO.read_line
        match line_res
          case Ok(line) then
            with IO
              IO.print("You said: ")
              IO.print_line(line)
              IO.exit(0)
            end
          case Error(_) then
            IO.exit(1)
        end
      end
    end

end
```

### Event-Driven `reactive program`

Long-running services, servers, and interactive apps use `reactive program`, exposing two total routines (`init` and `step`) that return `STEP[S]` (`Continue(next, cmds)` or `Stop(code, cmds)`):

```gardel
reactive program REACTIVE_HELLO

feature {ANY}

  init(args: SEQ[STRING]): STEP[NAT]
    do
      Continue(0, LIST.empty.prepend(Print("Hello. Press Ctrl-C to stop.")))
    end

  step(s: NAT, e: EVENT): STEP[NAT]
    do
      match e
        case Shutdown then Stop(0, LIST.empty.prepend(Print("Goodbye.")))
        case _ then Continue(s, LIST.empty)
      end
    end

end
```

---

## 11. Compiler CLI Quick Reference

| Command | Purpose |
| :--- | :--- |
| `gardelc file.gdl` | Verify contracts and emit C11/Boogie/Bend artifacts |
| `gardelc --project dir --strict` | Strict project verification (rejects any `assume`) |
| `gardelc --project dir --strict --run` | Verify strictly, compile, and run |
| `gardelc --inspect file.gdl` | Print public Eiffel-style `short` contract interface |
| `gardelc --project dir --json-diagnostics` | Emit structured diagnostics for automated repair loops |
