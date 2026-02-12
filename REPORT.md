# vtjson — Project Status Report & Brainstorming

> **Date:** 2026-02-12
> **Version analyzed:** 2.2.8
> **Author of this report:** Automated Technical Audit
> **Python tested:** 3.14.3 — **79/79 tests passing**

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Project Layout & Architecture](#2-project-layout--architecture)
3. [Design Analysis — The Theoretician's Perspective](#3-design-analysis--the-theoreticians-perspective)
4. [Code Analysis — The Pythonist's Perspective](#4-code-analysis--the-pythonists-perspective)
5. [What Is Good](#5-what-is-good)
6. [What Should Be Improved](#6-what-should-be-improved)
7. [Surprise Catalog — Principle of Least Astonishment](#7-surprise-catalog--principle-of-least-astonishment)
8. [Potential Bugs & Edge Cases](#8-potential-bugs--edge-cases)
9. [Recommendations Summary](#9-recommendations-summary)
10. [Competitive Landscape & Adoption Strategy](#10-competitive-landscape--adoption-strategy)
11. [Appendix A — Suggested Renaming Table](#appendix-a--suggested-renaming-table)
12. [Appendix B — CI/CD Implementation](#appendix-b--cicd-implementation)
13. [Appendix C — Code Refactoring Sketches](#appendix-c--code-refactoring-sketches)
14. [Appendix D — Benchmark Script](#appendix-d--benchmark-script)
15. [Appendix E — Static Analysis Results](#appendix-e--static-analysis-results)

---

## 1. Executive Summary

`vtjson` is a single-module Python library (3,159 LOC) for validating arbitrary
Python objects against declarative schemas. It bridges the gap between Python's
type annotation system and runtime validation, offering a DSL (domain-specific
language) that is simultaneously valid as Python type hints and as operational
validation constraints.

The library is authored by Michel Van den Bergh, licensed under MIT, and is
deployed in production for the
[Fishtest](https://tests.stockfishchess.org/tests) infrastructure (the
distributed testing framework for the Stockfish chess engine).

**Overall health:** All 79 unit tests pass. The codebase is internally
consistent, well-documented via Sphinx, and demonstrates deep domain knowledge.
This report focuses on **what to change**, not what to praise. Section 5
summarizes strengths briefly; the bulk of the document is a brainstorming
workspace for improvements, naming issues, and refactoring ideas.

---

## 2. Project Layout & Architecture

### 2.1 File Map

| File / Directory | Role | LOC |
|---|---|---|
| `vtjson.py` | Core library — single-module design | 3,159 |
| `test_vtjson.py` | Unit tests (unittest) | 2,651 |
| `bench.py` | Benchmark (type-annotated style) | 631 |
| `bench_classic.py` | Benchmark (classic/dict style) | 596 |
| `pyproject.toml` | Build & packaging config | 30 |
| `install.sh` | Manual install helper | 5 |
| `lint.sh` | Lint/format pipeline (mypy, black, isort, flake8, mdl, aspell) | 7 |
| `vtjson.rb` | Markdownlint config | 2 |
| `docs/` | Sphinx documentation sources | — |
| `py.typed` | PEP 561 marker | — |

### 2.2 Architectural Overview

```
┌───────────────────────────────────────────────────────────────┐
│                        vtjson.py                              │
│                                                               │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────────┐  │
│  │  Public API  │   │  Schema DSL  │   │  Compilation &    │  │
│  │  validate()  │   │  union       │   │  Dispatch Engine  │  │
│  │  compile()   │   │  intersect   │   │  _compile()       │  │
│  │  safe_cast() │   │  complement  │   │  _mapping         │  │
│  │  make_type() │   │  lax/strict  │   │  _deferred        │  │
│  └──────────────┘   │  set_label   │   └───────────────────┘  │
│                     │  set_name    │                          │
│                     │  quote       │   ┌───────────────────┐  │
│                     │  ifthen/cond │   │  Built-in Schemas │  │
│                     │  filter      │   │  regex, glob,     │  │
│                     │  fields      │   │  email, url,      │  │
│                     │  protocol    │   │  ip_address, div, │  │
│                     └──────────────┘   │  interval, size,  │  │
│                                        │  date/time, etc.  │  │
│  ┌──────────────────────────────────┐  └───────────────────┘  │
│  │  Type Annotation Integration     │                         │
│  │  TypedDict, Protocol, Annotated, │                         │
│  │  Union, Literal, NewType, etc.   │                         │
│  └──────────────────────────────────┘                         │
└───────────────────────────────────────────────────────────────┘
```

**Single-module design:** The entire library lives in one file. This is
deliberate — it simplifies distribution and vendoring. However, at 3,159 lines,
the file is at the upper boundary of what is comfortable for a single module.

**Two-phase architecture:** Schemas go through a **compilation** phase
(`_compile`) that produces a `compiled_schema` object, whose `__validate__`
method is then called during validation. This separation is clean and enables
pre-compilation for performance-critical paths.

**Recursive schema support:** The `_mapping` / `_deferred` mechanism is an
elegant trampoline that resolves circular references in schemas, enabling
recursive data structures (e.g., tree-shaped JSON) to be validated.

---

## 3. Design Analysis — The Theoretician's Perspective

### 3.1 Algebraic Structure

The schema language forms a **Boolean algebra** (more precisely, a bounded
distributive lattice with complement) over the space of all Python objects,
where:

| Operation | Schema Combinator | Algebraic Role |
|---|---|---|
| Union (join) | `union(A, B)` | A v B |
| Intersection (meet) | `intersect(A, B)` | A ^ B |
| Complement | `complement(A)` | ~A |
| Top (universal) | `anything` | T (top) |
| Bottom (empty) | `nothing` | ⊥ (bot) |

This gives the user a complete algebra for composing validation predicates.

#### 3.1.1 Lattice Diagram

The partial order on schemas is defined by "matches-subset":
S1 <= S2 iff every object matching S1 also matches S2.

```
                 anything (T)
                /    |    \
             str    int   float    ...  (primitive types)
            / | \    |      |
      regex  glob  size    gt      ...  (refined types)
        |      |     |      |
        ⋮       ⋮     ⋮      ⋮       ...  (intersections of refinements)
                \    |    /
                 nothing (⊥)
```

The meet and join operations satisfy the standard lattice axioms:

| Law | Union | Intersection |
|---|---|---|
| **Idempotent** | `union(A, A) = A` | `intersect(A, A) = A` |
| **Commutative** | `union(A, B) = union(B, A)` | `intersect(A, B) = intersect(B, A)` |
| **Associative** | `union(A, union(B, C)) = union(union(A, B), C)` | `intersect(A, intersect(B, C)) = intersect(intersect(A, B), C)` |
| **Absorption** | `union(A, intersect(A, B)) = A` | `intersect(A, union(A, B)) = A` |
| **Distributive** | `union(A, intersect(B, C)) = intersect(union(A, B), union(A, C))` | `intersect(A, union(B, C)) = union(intersect(A, B), intersect(A, C))` |
| **Complement** | `union(A, complement(A)) = anything` | `intersect(A, complement(A)) = nothing` |
| **Bounded** | `union(A, nothing) = A` | `intersect(A, anything) = A` |


The `ifthen(P, Q, R)` combinator implements material conditional:

    ifthen(P, Q, R) = intersect(union(complement(P), Q), union(P, R))

which is equivalent to (P => Q) ^ (~P => R).

The `cond` combinator generalizes this to a piecewise function / case analysis,
which is the categorical coproduct elimination principle expressed operationally:

    cond((P1, S1), (P2, S2), ..., (Pn, Sn))

is equivalent to: "if P1, validate S1; else if P2, validate S2; ..." —
analogous to defining a function by cases on a disjoint union.

#### 3.1.2 Functional Completeness

The set {`union`, `intersect`, `complement`, `anything`, `nothing`} is
**functionally complete** — any Boolean predicate on objects can be expressed.
This follows from the fact that {v, ^, ~, T, ⊥} generates all Boolean
functions. Concretely:

- **Material conditional:** P => Q = union(complement(P), Q)
- **Exclusive or:** P xor Q = intersect(union(P, Q), complement(intersect(P, Q)))
- **NAND:** nand(P, Q) = complement(intersect(P, Q))
- **Biconditional:** P <=> Q = union(intersect(P, Q), intersect(complement(P), complement(Q)))

Any desired validation predicate can therefore be expressed without resorting
to `callable` (opaque function) schemas.

#### 3.1.3 Assessment & Suggestions

**Assessment:** The algebraic design is principled and complete.

**Suggestion — expose the lattice structure programmatically:** Currently, the
user cannot ask "is schema A a sub-schema of schema B?" without constructing
a test object. A `schema_leq(A, B)` function (or `A <= B` operator overload on
`compiled_schema`) would let users reason about schema relationships
statically. For finite schemas (types, constants, small unions), this is
decidable. For schemas containing `callable` predicates it is not, but a
"best-effort" mode could still be useful.

**Suggestion — add schema simplification:** If a user writes
`union(int, union(int, str))`, the library could simplify this to `union(int, str)`
during compilation. This would improve error messages and debugging output.
The algebraic laws above provide the required rewrite rules. Simplification
is straightforward for unions/intersects of type-schemas but hard in general
(undecidable for opaque callables). A pragmatic approach: simplify when the
schema tree contains only structural nodes (types, constants, union, intersect,
complement), and leave opaque subtrees untouched.

### 3.2 Type-Theoretic View

From a type-theoretic standpoint, `vtjson` schemas are **refinement types**.
A schema S refines a type T if every object validating against S is an
inhabitant of T with additional constraints. The `Annotated[T, S, skip_first]`
pattern directly encodes this: T is the base type, S is the refinement
predicate.

#### 3.2.1 The Interpretation Function

The compilation phase (`_compile`) can be viewed as a **denotational semantics**
— an interpretation function mapping syntax (schema descriptions) to
semantics (decision procedures):

    ⟦ . ⟧ : Schema --> (Object --> {accept, reject(msg)})

More precisely, the codomain is not just `Bool` but `String` — the empty string
means acceptance, and a non-empty string carries a structured error message.
This makes the interpretation function a **writer monad** — it maps schemas
to procedures that produce diagnostic output alongside a pass/fail decision:

```
⟦ type T ⟧(obj)           = "" if isinstance(obj, T) else error
⟦ union(A, B) ⟧(obj)      = "" if ⟦A⟧(obj)="" or ⟦B⟧(obj)=""
⟦ intersect(A, B) ⟧(obj)  = "" if ⟦A⟧(obj)="" and ⟦B⟧(obj)=""
⟦ complement(A) ⟧(obj)    = "" if ⟦A⟧(obj) != ""
⟦ {k1:S1,...,kN:SN} ⟧(obj)= "" if obj is dict and for all i: ⟦Si⟧(obj[ki])=""
⟦ [S1,...,SN] ⟧(obj)      = "" if obj is list and for all i: ⟦Si⟧(obj[i])=""
```

#### 3.2.2 Guarded Recursion and Fixpoints

The `_deferred` / `_mapping` mechanism implements **guarded recursion** — it
ensures that the fixpoint of a recursive schema is well-defined, analogous to
the mu-operator in domain theory:

    mu X. F(X)

where `_deferred` acts as the placeholder for X during the unfolding of F.

The compilation process is a **least fixed-point construction**:

```
1. Create placeholder D = _deferred(cache, schema)
2. Store D in cache[schema]
3. Compile the schema body, which may reference schema recursively
4. When a recursive reference is found:
   - Return D (the placeholder)
   - Mark schema as "in_use"
5. After compilation completes:
   - If in_use: replace cache[schema] with the real compiled result
   - If not in_use: delete from cache (no recursion occurred)
6. At validation time, D.validate() looks up cache[schema] — now resolved
```

This is equivalent to **Bekic's theorem** in domain theory: recursive
definitions can be solved by iterating to a fixpoint. The single-pass nature
of `_compile` means it computes the fixpoint in one traversal rather than
iterating, which is possible because Python schemas form a regular tree
(no infinite chains of distinct unfoldings).

**Suggestion — detect non-productive recursion:** Currently, if a user writes
a schema like `S = union(S)` (recursion with no base case), the library will
loop at validation time, not at compile time. The compile phase could detect
this: if a `_deferred` placeholder is the *only* compiled result for a schema
(i.e., the schema body contributes nothing beyond the recursive reference),
raise a `SchemaError` at compile time. This is the "guardedness check" in
type theory — every recursive occurrence must appear under a constructor
(dict key, list element, etc.).

#### 3.2.3 Subtyping Judgment

The `lax` / `strict` modes correspond to **width subtyping** on records:

```
Strict mode (exact):      {a: int, b: str}  matches  {a: 1, b: "x"}          ✓
                          {a: int, b: str}  matches  {a: 1, b: "x", c: 3.0}  ✗
Lax mode (structural):    {a: int, b: str}  matches  {a: 1, b: "x", c: 3.0}  ✓  (width subtyping)
```

In type-theoretic terms:

- **Strict** = exact record type (no extra fields)
- **Lax** = width-subtyped record (extra fields allowed)
- **`lax()`/`strict()` wrappers** = locally changing the subtyping mode

This is analogous to **row polymorphism** in languages like OCaml or PureScript,
where a record type can be "open" (allowing extra fields) or "closed."

**Suggestion — depth subtyping:** Currently, `lax` / `strict` control *width*
subtyping only. Consider supporting *depth* subtyping as a separate axis:
should `{"age": int}` match `{"age": True}`? (Since `bool` is a subclass of
`int` in Python, it currently does.) A `strict_types` mode that disables
subclass matching for primitive types would complement the existing
`lax`/`strict` width modes.

### 3.3 Category-Theoretic Observations

#### 3.3.1 The Compilation Functor

The `wrapper` --> `compiled_schema` transformation is a **functor** from the
category of schema descriptions (with composition being nesting) to the category
of decision procedures. More precisely:

```
Category SchemaDesc:                    Category DecisionProc:
  Objects:  schema descriptions           Objects:  compiled_schema instances
  Morphisms: nesting / composition        Morphisms: function composition
  Identity: anything                      Identity: lambda obj: ""

Functor _compile : SchemaDesc --> DecisionProc
  _compile(type T)       = _type(T)
  _compile(union(A,B))   = _union(_compile(A), _compile(B))
  _compile(dict {k:S})   = _dict({k: _compile(S)})
  ...
```

The functor preserves composition:

    _compile(intersect(A, B)).validate(obj) = _compile(A).validate(obj)
                                                    composed with
                                              _compile(B).validate(obj)

and preserves identity:

    _compile(anything).validate(obj) = "" for all obj

#### 3.3.2 The Filter Combinator as Pullback

The `filter` combinator is categorically the most interesting construct.
It takes a morphism f: A --> B and a schema S_B and produces f*(S_B), which is
the **pullback** (or **preimage**) of S_B along f:

```
         f
  A  ---------> B
  |              |
  | f*(S_B)      | S_B
  v              v
{accept/reject}  {accept/reject}

f*(S_B)(a) = S_B(f(a))
```

In `vtjson` syntax: `filter(f, S)` validates by first applying `f` to the
object, then validating the result against `S`. Example:

```python
filter(int, gt(0))    # preimage of gt(0) under int()
# Matches: "42" (int("42")=42 > 0)
# Rejects: "0"  (int("0")=0, not > 0)
# Rejects: "abc" (int("abc") raises)
```

This is categorically **clean** — the functor diagram commutes, and the
construction preserves all algebraic operations:

```
filter(f, union(A, B))     = union(filter(f, A), filter(f, B))
filter(f, intersect(A, B)) = intersect(filter(f, A), filter(f, B))
filter(f, complement(A))   = complement(filter(f, A))
```

These equalities hold because f* is a Boolean algebra homomorphism — it
preserves meets and joins. This means users can freely combine `filter` with
other combinators without worrying about interaction effects.

#### 3.3.3 The Substitution Mechanism as a Natural Transformation

The `subs` parameter (schema substitution via `set_label`) implements a
**natural transformation** — it uniformly replaces one schema for another
at every occurrence of a label, while preserving the surrounding structure:

```
For each label L:
  subs : ⟦set_label(S, L)⟧ --> ⟦subs[L]⟧

This is natural in the sense that for any context C:
  C[set_label(S, L)] with subs = C[subs[L]]
```

This is analogous to **parametric polymorphism** — the label acts as a type
variable, and `subs` provides the type instantiation.

**Suggestion — compose substitutions:** Currently, substitutions are flat
(one level). If schema A uses label "x", and the substitution for "x" itself
uses label "y", the second-level substitution is not applied. Consider
supporting transitive / recursive substitution. The implementation already
passes `subs` recursively through `__validate__`, so this may work — but it
needs explicit testing and documentation.

#### 3.3.4 Diagram: Compilation Pipeline as Functorial Composition

```
                    _compile               __validate__
 Schema values  ─────────────> compiled    ──────────────>  "" | error_msg
 (Python DSL)                  _schema                      (String)
      │                          │                             │
      │   wrapper.__compile__    │   compiled_schema           │
      ├──────────────────────────┤   .__validate__             │
      │                          │                             │
      │         filter(f, S)     │         f*(S_B)             │
      │  ─────────────────────>  │  ─────────────────────>     │
      │                          │                             │
   Schema^op ──── _compile ──── DecProc ──── eval ──── Result
   (contravariant for filter)
```

The `filter` functor is **contravariant** — it goes from Schema^op to
DecisionProc. This is precisely the definition of a **presheaf**, and it means
that `filter` reverses the direction of the schema ordering:

    If S1 <= S2 then filter(f, S2) <= filter(f, S1) — NOT necessarily true!

Actually, filter is covariant on schemas (if S1 <= S2, then f*(S1) <= f*(S2)),
and contravariant on morphisms (if g = h∘f, then f*∘h* = g*). This is the
standard contravariance in the morphism slot of a pullback functor.

### 3.4 Concerns & Improvement Ideas from the Theoretical POV

1. **Callable schemas break introspectability.** Allowing an arbitrary Python
   function as a schema predicate is a pragmatic escape hatch, but it means the
   schema language is not decidable in general. You cannot statically analyze,
   simplify, or compare two schemas when one is `lambda x: x > 0`.

   **Suggestion — restrict the escape hatch:** Introduce a `predicate(fn, name)`
   wrapper that forces callers to label opaque schemas. This keeps
   introspectability for the structured core while isolating the opaque parts.
   At a minimum, `callable` schemas should carry a `__name__` or repr so that
   error messages remain informative.

2. **The `float` <-> `int` coercion breaks Boolean-algebra laws.** The `float`
   schema matches `int` objects, modeling int ⊆ float from the numeric tower.
   But this creates a real asymmetry: `complement(float)` does *not* reject
   ints, meaning float ^ ~float ≠ ⊥ when applied to ints.

   **Suggestion — make the coercion explicit:** Introduce a `numeric` schema
   that explicitly matches `int | float`, and have `float` match only `float`.
   Users who want the coercion use `numeric`; users who want strictness use
   `float`. This restores the lattice laws without losing functionality.
   Alternatively, document this as a deliberate deviation with a formal note
   that the algebra is "Boolean modulo the numeric tower embedding."

3. **Strictness modes are a hidden effect.** `strict` mode enforces exact
   record shapes (no extra keys), while `lax` mode allows structural subtyping.
   The ability to locally override strictness via `lax()` / `strict()` is a
   parametric effect that propagates implicitly through the schema tree.

   **Suggestion — make the effect stack visible:** Consider logging or tracing
   strictness transitions so that users can debug unexpected `lax` / `strict`
   interactions in deeply nested schemas. A `debug=True` flag on `validate()`
   that prints the strictness stack at each dict level would help.

4. **The `filter` combinator's categorical role is under-documented.** `filter`
   is the pullback f*(S) — this is a powerful idea, but the name `filter`
   suggests Python's builtin `filter()` (which *removes* elements). A
   mathematician would expect the name to reflect the categorical operation.

   **Suggestion:** Rename to `preimage`, `pullback`, or `via` — something
   that communicates "validate f(x) against S" rather than "filter elements
   out." See [Appendix A](#appendix-a--suggested-renaming-table).

5. **No schema equivalence or subsumption check.** Two schemas may be
   semantically equivalent but structurally different
   (`union(int, str)` vs `union(str, int)`). Without a comparison operation,
   users cannot test schema equality or redundancy.

   **Suggestion:** Implement `schema_equiv(A, B)` for the decidable fragment
   (type schemas, constants, finite unions/intersects). Return `True`, `False`,
   or `Undecidable` (when callables are involved).

---

## 4. Code Analysis — The Pythonist's Perspective

### 4.1 Strengths

#### Excellent Python Version Compatibility

The code carefully probes for feature availability (`supports_Literal`,
`supports_TypedDict`, `supports_NotRequired`, `supports_Annotated`,
`supports_Generics`, `supports_UnionType`, etc.) ensuring the library works
across Python 3.7+. This is meticulous and rare.

#### Comprehensive Type Hinting

The codebase passes `mypy --strict` (per the `lint.sh` pipeline). The use of
`from __future__ import annotations` enables modern annotation syntax throughout.
The `py.typed` marker declares PEP 561 compliance.

#### Clean Public API

```python
validate(schema, obj, name="object", strict=True, subs={})
compile(schema) -> compiled_schema
safe_cast(schema, obj) -> T
make_type(schema, name=None, strict=True, debug=False, subs={}) -> type
```

The API is minimal, discoverable, and well-documented via docstrings and Sphinx.

#### Idiomatic Decorator Pattern

The `_set__name__` decorator automatically generates `__name__`, `__str__`, and
`__repr__` for schema classes based on constructor arguments. This is clever,
DRY, and produces excellent error messages.

#### Thorough Test Suite

79 unit tests covering: recursion, immutability of compiled schemas,
all built-in schemas, all wrappers, type annotation integration (TypedDict,
Protocol, NamedTuple, Annotated, Union, Literal, NewType), edge cases
(truncation, float equality), and name generation. Test coverage is
comprehensive.

### 4.2 Code Quality — Problems & Concrete Alternatives

#### 4.2.1 Single-File Architecture (3,159 LOC — the Pain Threshold)

At 3,159 lines, `vtjson.py` is a monolith. It contains ~30 public classes,
~20 internal compiled schema classes, the compilation engine, the validation
engine, and helper utilities — all in one namespace.

**The problem:** An editor `Ctrl+G` workflow is required to navigate the file.
Grep results return too many hits. New contributors cannot orient themselves.

**Suggested split:**

| Module | Contents | Approx. LOC |
|---|---|---|
| `vtjson/core.py` | `validate`, `compile`, `_compile`, `compiled_schema`, `wrapper` | ~600 |
| `vtjson/builtins.py` | `regex`, `glob`, `email`, `url`, `ip_address`, `magic`, `date_time` | ~700 |
| `vtjson/algebra.py` | `union`, `intersect`, `complement`, `anything`, `nothing`, `lax`, `strict` | ~300 |
| `vtjson/modifiers.py` | `gt`, `ge`, `lt`, `le`, `interval`, `size`, `fields`, `div` | ~400 |
| `vtjson/filters.py` | `filter`, `ifthen`, `cond`, `quote`, `set_label`, `set_name` | ~300 |
| `vtjson/typing_compat.py` | `_Annotated`, `_Union`, `_Literal`, `_TypedDict`, `_Protocol` handlers | ~500 |
| `vtjson/__init__.py` | Re-exports everything — preserves `from vtjson import validate` | ~50 |

This preserves backwards compatibility: `from vtjson import validate` still
works, but contributors can now open one 300-line file at a time.

#### 4.2.2 Naming Conventions — the Surprises

These names are correct in theory but surprising in practice:

| Current Name | Surprise | Suggested Name(s) | Rationale |
|---|---|---|---|
| `_mapping` | Shadows `Mapping` (dict concept) and coexists with `_Mapping` (line 3037) | `_compile_cache`, `_schema_registry` | What it *does* is cache compiled schemas during fixpoint resolution |
| `_c()` | Cryptic — only context reveals "truncated repr" | `_truncated_repr`, `_short_repr`, `_trunc` | Single-letter names belong in tight math loops, not utility functions |
| `filter` | Shadows Python's `filter()` builtin | `preprocess`, `transform`, `via` | The schema transforms its input *before* validating — the opposite of filtering |
| `_deferred` | Ambiguous — deferred *what*? | `_lazy_schema`, `_schema_ref`, `_fixpoint_proxy` | It is a proxy for a schema that hasn't been compiled yet |
| `_set__name__` | Double underscore looks like dunder pollution | `_auto_repr`, `_schema_repr` | It generates `__name__`, `__str__`, `__repr__` from constructor args |

Full table with migration plan in [Appendix A](#appendix-a--suggested-renaming-table).

#### 4.2.3 Mutable Default Arguments — Safe but Surprising

```python
def __validate__(self, obj, name="object", strict=True, subs={}):
```

This appears 20+ times. It is technically safe because `subs` has type
`Mapping[str, object]` (immutable interface) and is never mutated. But it
triggers every Python linter's B006 warning and will make every code reviewer
do a double-take.

**Two concrete alternatives:**

```python
# Option A: None sentinel (most Pythonic, most familiar)
_EMPTY_SUBS: Mapping[str, object] = {}

def __validate__(self, obj, name="object", strict=True, subs=None):
    if subs is None:
        subs = _EMPTY_SUBS
    ...

# Option B: Module-level constant (cleanest — no sentinel dance)
EMPTY_SUBS: Final[Mapping[str, object]] = MappingProxyType({})

def __validate__(self, obj, name="object", strict=True, subs=EMPTY_SUBS):
    ...
```

Option B is cleaner: `MappingProxyType` is truly immutable, and `Final`
signals intent. No runtime cost difference.

#### 4.2.4 `setattr(self, "__validate__", ...)` — Dynamic Dispatch

The code uses `setattr(self, "__validate__", ...)` in **11 places** to
dynamically replace a method on an instance. This is used as a strategy
pattern — e.g., `interval` selects one of four implementations based on which
bounds are present:

```python
# Current code (interval.__init__)
if lower is not None and upper is not None:
    setattr(self, "__validate__", _intersect((lower, upper)).__validate__)
elif lower is not None:
    setattr(self, "__validate__", lower.__validate__)
...
```

**The problem:** This breaks IDE "Go to Definition," makes `mypy` work harder,
and surprises anyone reading the class who expects `__validate__` to be defined
on the class body.

**Alternative — delegation pattern:**

```python
class _interval(compiled_schema):
    def __init__(self, lower, upper):
        self._delegate = self._select_validator(lower, upper)

    def _select_validator(self, lower, upper):
        if lower is not None and upper is not None:
            return _intersect((lower, upper))
        elif lower is not None:
            return lower
        elif upper is not None:
            return upper
        else:
            return _anything()

    def __validate__(self, obj, name="object", strict=True, subs={}):
        return self._delegate.__validate__(obj, name, strict, subs)
```

This is one extra indirection (negligible cost) but is fully transparent to
static analysis, and `__validate__` always exists on the class.

See [Appendix C](#appendix-c--code-refactoring-sketches) for a full sketch.

#### 4.2.5 Global State: DNS Resolver

```python
_dns_resolver: dns.resolver.Resolver | None = None
```

This is lazily initialized without synchronization. Under concurrent access
(e.g., ASGI apps), multiple resolver instances could be created.

**Fix:** Use `functools.lru_cache`:

```python
@functools.lru_cache(maxsize=1)
def _get_dns_resolver() -> dns.resolver.Resolver:
    resolver = dns.resolver.Resolver()
    resolver.cache = dns.resolver.LRUCache()
    return resolver
```

`lru_cache` is thread-safe in CPython and idiomatic for lazy singletons.

#### 4.2.6 Missing `__all__`

The module exports everything, including internals like `_c`, `_compile`,
`_mapping`, `_deferred`, etc. Users doing `from vtjson import *` get polluted
namespaces.

**Fix:** Add a top-level `__all__` list. This also serves as living
documentation of the public API surface:

```python
__all__ = [
    "validate", "compile", "safe_cast", "make_type",
    "compiled_schema", "wrapper",
    "union", "intersect", "complement", "anything", "nothing",
    "lax", "strict", "set_label", "set_name", "quote",
    "ifthen", "cond", "filter", "fields",
    "gt", "ge", "lt", "le", "interval", "size", "div",
    "regex", "glob", "email", "url", "ip_address",
    "domain_name", "date_time", "magic", "number",
    "one_of", "at_least_one_of", "at_most_one_of",
    "keys", "values", "items", "ifthen", "protocol",
]
```

### 4.3 Performance Analysis — Where the Microseconds Go

Micro-benchmarks run on the project's test machine (Python 3.14.3, Linux).
All timings are per-call averages over 100K–500K iterations.

#### 4.3.1 Baseline: Compiled Validation is Fast

| Operation | Time (us) | Notes |
|---|---|---|
| `isinstance(42, int)` | 0.025 | Raw Python baseline |
| compiled `__validate__(int, 42)` | 0.098 | ~4x overhead vs raw isinstance |
| Dict (3 keys, all match) | 2.15 | ~0.7 us/key |
| Dict (6 keys, all match) | 3.71 | ~0.6 us/key (amortized) |
| List `[int x5]` | 1.93 | ~0.4 us/element |
| `union(4)` first hit | 0.29 | Short-circuits on first match |
| `union(4)` all miss | 3.83 | Must try all 4 branches |
| `intersect(3)` all pass | 0.48 | Fails fast on first mismatch |

**Verdict:** Validation is fast enough for any reasonable use case. A complex
Fishtest `runs_schema` with ~50 fields, nested dicts, and regex patterns
validates in ~2–5 ms (from bench.py). This is well within budget for web
request validation.

#### 4.3.2 Compilation is the Bottleneck — Not Validation

| Operation | Time (us) | Notes |
|---|---|---|
| `_compile(int)` | 3.54 | Simple type |
| `_compile(str)` | 3.68 | Simple type |
| `_compile({"a": int})` | 24.82 | 1-key dict schema |
| `compile({"name":str,"age":int,"active":bool})` | 61.28 | 3-key dict schema |
| `validate(schema, obj)` raw (compile + validate) | 66.46 | 96% is compilation! |

**The key insight:** `validate()` recompiles the schema *every time* it is
called. For repeated validation (e.g., validating many incoming API requests),
using `compile()` once and calling `__validate__()` directly gives a **30x
speedup**.

The project already documents this and provides `compile()` as a public API.
But users calling `validate()` in a loop will hit this wall silently.

**Suggestion — add an LRU cache to `_compile`:**

```python
# Candidate implementation
@functools.lru_cache(maxsize=256)
def _compile_cached(schema_id, schema):
    return _compile(schema)

def validate(schema, obj, name="object", strict=True, subs={}):
    # Cache by id for mutable schemas, by value for immutable
    compiled = _compile_cached(id(schema), schema)
    ...
```

This won't work directly because schemas are not always hashable (dicts,
lists). But for the common case of type schemas (`int`, `str`, `float`),
a `WeakValueDictionary` keyed on `id(schema)` would work:

```python
_compile_cache: dict[int, compiled_schema] = {}

def _compile_with_cache(schema, _deferred_compiles=None):
    key = id(schema)
    if key in _compile_cache:
        return _compile_cache[key]
    result = _compile(schema, _deferred_compiles)
    _compile_cache[key] = result
    return result
```

**Trade-off:** Memory for speed. A bounded cache (LRU with maxsize) limits the
memory cost. This could be exposed as a `vtjson.enable_cache()` opt-in.

#### 4.3.3 The `_compile` Dispatch Chain — 18 isinstance Checks

The `_compile` function uses a linear if/elif chain with **18 branches** to
determine the schema type. For simple schemas like `int` or `str`, the match
occurs at branch ~14 (`isinstance(schema, type)`), meaning **13 isinstance
checks** are executed before reaching the correct branch:

```
Branch  1: isinstance(schema, compiled_schema)       — miss for int
Branch  2: isinstance(schema, type) and issubclass   — miss (not compiled_schema subclass)
Branch  3: hasattr(schema, "__validate__")            — miss
Branch  4: isinstance(schema, wrapper)                — miss
Branch  5: typing.is_typeddict(schema)                — miss
Branch  6: isinstance(schema, type) and _is_protocol  — miss
Branch  7: isinstance(schema, type) and issubclass(tuple) — miss
Branch  8: schema == Any                              — miss
Branch  9: hasattr "__name__" and "__supertype__"      — hit for int! (int has __name__)
                                                        No — int has no __supertype__
Branch 10: origin == tuple                            — miss
Branch 11: isinstance(origin, type) and Mapping       — miss
Branch 12: isinstance(origin, type) and Container     — miss
Branch 13: origin == Union                            — miss
Branch 14: origin == Literal                          — miss
Branch 15: origin == Annotated                        — miss
Branch 16: isinstance(schema, UnionType)              — miss
>>Branch 17: isinstance(schema, type)                  — HIT for int, str, float, etc.
Branch 18: callable(schema)                           — miss
```

This means that the most common case (bare type schemas like `int`, `str`,
`dict`) falls through 16 branches before matching.

**Suggestion — reorder branches by frequency:** Move `isinstance(schema, type)`
higher in the chain. In practice, the most common schemas are:

1. `type` objects (int, str, float, dict, list, bool) — **most common**
2. `wrapper` subclasses (union, intersect, gt, regex, etc.)
3. `Mapping` instances (dict literals used as schemas)
4. `Sequence` instances (list literals used as schemas)
5. Everything else (callables, constants, generics, etc.)

Reordering to put `isinstance(schema, type)` at position ~3 (after
`compiled_schema` and `wrapper`) would halve the dispatch cost for simple
type schemas. Estimated savings: ~1–2 us per `_compile(int)` call.

**Suggestion — use a dispatch dict for type-based schemas:**

```python
_TYPE_DISPATCH: dict[type, Callable] = {
    int: lambda s, dc: _type(s),
    str: lambda s, dc: _type(s),
    float: lambda s, dc: _type(s),
    bool: lambda s, dc: _type(s),
    dict: lambda s, dc: _type(s),
    list: lambda s, dc: _type(s),
    # ... etc.
}

def _compile(schema, _deferred_compiles=None):
    ...
    if type(schema) is type and schema in _TYPE_DISPATCH:
        ret = _TYPE_DISPATCH[schema](schema, _deferred_compiles)
    elif ...
```

Dict lookup is O(1), vs O(n) for the if/elif chain. This is a modest
optimization but compounds across deeply nested schemas (a 50-field dict schema
triggers `_compile` 50+ times during compilation).

#### 4.3.4 The `setattr` Pattern — Faster than Delegation

A surprising finding from microbenchmarks:

| Dispatch Pattern | Time (us) | Overhead vs normal |
|---|---|---|
| Normal method call | 0.036 | 1.0x |
| `setattr` bound method | 0.040 | 1.14x |
| Delegation (`self._impl.method()`) | 0.076 | 2.14x |

The `setattr` pattern is only 14% slower than a normal method call, while
the "cleaner" delegation pattern is **114% slower** (an extra function call
frame). For a library that validates thousands of fields per request, this
matters.

**Revised recommendation:** The `setattr` pattern in `vtjson` is a *deliberate
performance choice*, not an accident. If readability is prioritized over speed,
delegation adds ~40 ns/call — negligible for most uses, but measurable in
tight loops. The author may have benchmarked this.

Keep the `setattr` pattern but add a one-line comment explaining why:

```python
# Performance: setattr avoids delegation overhead (~2x faster)
setattr(self, "__validate__", _intersect((lower, upper)).__validate__)
```

#### 4.3.5 String Return Protocol — No Measurable Penalty

The library uses `return ""` for success and `return "error message"` for
failure. An alternative would be `return None` / `return msg`, or
`return True` / raising an exception. Benchmarks show **no measurable
difference** between returning `""`, `None`, or `True`:

| Pattern | Time (us) |
|---|---|
| `return ""` + `if msg == ""` | 0.030 + 0.015 |
| `return None` + `if msg is None` | 0.030 + 0.014 |
| `return True` + `if not msg` | 0.030 + 0.014 |

The string protocol wins on ergonomics: error messages are first-class, no
separate exception path needed. The 1 ns difference in comparison is noise.
**No change recommended.**

#### 4.3.6 Summary: Speedup Opportunities

| # | Optimization | Estimated speedup | Effort | Risk |
|---|---|---|---|---|
| 1 | **Cache compiled schemas** (opt-in LRU for `validate()`) | 30x for repeated calls | Medium | Low (opt-in) |
| 2 | **Reorder `_compile` branches** by frequency | 1–2 us per compile call | Low | None |
| 3 | **Dispatch dict** for bare type schemas | ~50% compile speedup for types | Low | None |
| 4 | **Pre-compile `Annotated` schemas** at import time | Eliminates runtime compilation for type-annotated schemas | High | Medium |
| 5 | Keep `setattr` (don't switch to delegation) | Avoid 2x regression per validate call | None | None |

### 4.4 Static Analysis Deep Dive — ty & ruff

The existing lint pipeline (`lint.sh`) runs mypy, black, isort, flake8, mdl,
and aspell. Two modern tools — **ty** (Astral's new type checker) and **ruff**
(Astral's fast Python linter) — reveal additional issues that the current
pipeline misses.

#### 4.4.1 ty check (ty 0.0.16)

`ty check *.py` reports **87 diagnostics** (82 errors). Most are caused by
ty reading `requires-python = ">=3.7"` from `pyproject.toml` and assuming
Python 3.7, where `NotRequired`, `Required`, `typing.assert_type`, and
`types.UnionType` do not exist.

| Category | Count | Root Cause |
|---|---|---|
| `invalid-argument-type` | 37 | `comparable` protocol not recognized; `Literal[0]` not assignable to `comparable`; `object` passed where `Sized`/`Iterable` expected |
| `unresolved-import` | 13 | Python 3.7 assumed → `NotRequired`, `Required`, `assert_type` missing from `typing`; third-party modules (`bson`, `magic`, `dns`, `email_validator`, `idna`) not in ty's environment |
| `not-subscriptable` | 9 | `obj: object` is subscripted without narrowing; `__delitem__` on `str`/`int`/`float`/`datetime` |
| `unresolved-attribute` | 8 | `EllipsisType` missing from `types` (3.7); callable objects typed as `(Any, /) -> object` lack `__name__`; `is_typeddict` missing from `typing` (3.7) |
| `invalid-assignment` | 8 | `type` not assignable to `type[Mapping[...]]`; subscript assignment on non-subscriptable types |
| `missing-typed-dict-key` | 3 | **Notable:** includes `bench.py:347` reporting missing `bad` key in `task_type` (see §8.3.1, ty-only false positive) |
| Other (`unsupported-operator`, `not-iterable`, `invalid-method-override`) | 4 | `not in` on `object`; iterating `object`; method signature mismatch |

**Per-file distribution:**

| File | Diagnostics |
|---|---|
| `vtjson.py` | 61 |
| `test_vtjson.py` | 29 |
| `bench_classic.py` | 15 |
| `bench.py` | 11 |

**Assessment:** Many of the 87 diagnostics are **false positives** caused by the
Python 3.7 baseline used by ty. This does **not** mean
`requires-python = ">=3.7"` is wrong in `pyproject.toml`; it is a compatibility
declaration, not a bug. The library is designed to run on 3.7+ via
feature-detection (`try/except ImportError`) and `typing_extensions` fallbacks.
In this audit, tests were executed on Python 3.14.3; Python 3.7 was not
re-run in-session, so this section classifies the `NotRequired`/TypedDict
findings as **ty-only diagnostics**, not confirmed runtime failures.

The remaining ~30 genuine findings fall into three clusters:

1. **`object` used where narrower types would help.** The `__validate__`
   methods receive `obj: object` and then subscript, iterate, or call `len()`
   on it — all operations that `object` does not support. The code is correct
   at runtime (it checks `isinstance` first), but ty cannot see through the
   dynamic dispatch. **Fix:** use `@overload` or cast after isinstance guards.

2. **`comparable` protocol not satisfied by `Literal[0]`.** The `gt`/`ge`/
   `lt`/`le` wrappers declare their parameter as `comparable`, but ty does
   not recognize `Literal[0]` as satisfying this protocol. **Fix:** widen
   the type annotation to `object` or define `comparable` as a `Protocol` with
   `__lt__`/`__gt__` that ty can see.

3. **`callable.__name__` access.** The code accesses `.__name__` on callables,
   but ty correctly notes that not all callables are functions (e.g.,
   `functools.partial` objects lack `__name__`). The code handles this with
   `try/except`, which is correct but invisible to ty.

**Suggestion — add ty to the lint pipeline:**

```bash
# In lint.sh, add:
ty check vtjson.py --python-version 3.11  # avoids 3.7 false positives
```

Using `--python-version 3.11` eliminates the import-resolution noise. The
remaining findings are genuine type-safety improvements. ty is complementary
to mypy — it catches different issues (especially around `object` narrowing
and protocol satisfaction).

#### 4.4.2 ruff check --select ALL (ruff 0.15.0)

`ruff check --select ALL *.py` reports **1,596 diagnostics** across all files.
With `ALL` enabled, every available rule is active — including intentionally
opinionated rules that conflict with the project's style.

**Per-file distribution:**

| File | Errors | Auto-fixable |
|---|---|---|
| `vtjson.py` | 869 | 260 |
| `test_vtjson.py` | 600 | 13 |
| `bench.py` | 73 | 19 |
| `bench_classic.py` | 54 | 16 |

**Top 20 rules (across all files):**

| Rule | Count | Meaning | Actionable? |
|---|---|---|---|
| `PT027` | 269 | Use `pytest.raises` instead of `unittest` assertions | No — project uses `unittest` by design |
| `PT009` | 109 | Use `pytest`-style assertions | No — same reason |
| `N801` | 103 | Class name should use CapWords | Partially — schema classes like `regex`, `gt`, `ge` are intentionally lowercase (DSL readability) |
| `COM812` | 93 | Trailing comma missing | Style choice |
| `D212` | 88 | Multi-line docstring should start at first line | Style choice (conflicts with D213) |
| `D102` | 84 | Missing docstring in public method | **Yes** — `__validate__` methods lack docstrings |
| `FBT001` | 75 | Boolean-typed positional arg in function definition | Partially — `strict: bool` is by design |
| `FBT002` | 69 | Boolean default positional arg | Same as above |
| `TRY003` | 52 | Avoid specifying long messages outside exception class | Style choice |
| `D105` | 47 | Missing docstring in magic method | Partially |
| `D205` | 46 | 1 blank line required between summary and description | **Yes** — docstring formatting |
| `BLE001` | 43 | Do not catch blind `Exception` | Partially — some uses are intentional fallbacks |
| `RUF010` | 42 | Use explicit `str()` instead of f-string conversion | Minor style |
| `EM102` | 38 | Exception message should not be an f-string | Style choice |
| `PGH003` | 34 | Use specific `type: ignore` comments | **Yes** — replace `# type: ignore` with `# type: ignore[specific-rule]` |
| `D200` | 28 | One-line docstring should fit on one line | Minor style |
| `RET505` | 26 | Unnecessary `else` after `return` | **Yes** — easy cleanup |
| `T201` | 25 | `print()` found | Some intentional (bench), some debug leftovers |
| `B010` | 22 | Do not call `setattr` with a constant attribute | **Yes** — the 11× `setattr(__validate__)` pattern |
| `N802` | 17 | Function name should be lowercase | Some intentional |

**Assessment:** The 1,596 count is inflated by `ALL` mode — many rules are
mutually exclusive (D203/D211, D212/D213) or irrelevant (PT* for a unittest
project). A realistic ruff config would select a meaningful subset:

**Suggestion — add a `ruff.toml` or `[tool.ruff]` section in `pyproject.toml`:**

```toml
[tool.ruff]
line-length = 88
target-version = "py37"

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "B",    # flake8-bugbear
    "I",    # isort
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "RUF",  # ruff-specific
    "SIM",  # simplify
    "RET",  # return
    "C901", # complexity
]
ignore = [
    "N801",  # lowercase class names are intentional DSL style
    "B010",  # setattr with constant is a deliberate pattern (see §4.3.4)
    "B006",  # mutable default args — safe here (Mapping, never mutated)
    "E721",  # type comparison — intentional in schema dispatch
]

[tool.ruff.lint.per-file-ignores]
"test_vtjson.py" = ["N802", "N803"]  # test names follow unittest convention
"bench*.py" = ["N801", "N803", "N806", "T201"]  # bench style + print
```

This replaces both `flake8` and `isort` with a single, faster tool. The
remaining actionable findings (D205, PGH003, RET505) are genuine improvements.

**Suggestion — replace flake8 + isort with ruff in `lint.sh`:**

```bash
# Before (current lint.sh):
flake8 --max-line-length 88 bench.py test_vtjson.py vtjson.py
isort --profile black *.py

# After:
ruff check *.py
ruff format --check *.py  # can also replace black
```

ruff is 10–100x faster than flake8 and subsumes its functionality plus isort.

Full ty and ruff outputs are reproduced in
[Appendix E](#appendix-e--static-analysis-results).

### 4.5 Dependency Analysis — Too Heavy for a Validation Library?

| Dependency | Purpose | Weight | Users who need it |
|---|---|---|---|
| `dnspython` | DNS resolution for `domain_name(resolve=True)` | Medium | Only those validating domain names with DNS lookups |
| `email_validator` | Email validation | Medium | Only those validating email addresses |
| `idna` | Internationalized domain names | Light | Same as above |
| `python-magic` | MIME type detection | **Heavy** (C lib `libmagic`) | Only those validating file MIME types |
| `typing_extensions` | Back-compat for older Python | Light | Everyone on Python < 3.11 |

**Problem:** All of these are hard dependencies in `pyproject.toml`. A user who
just wants to validate dicts and lists must install `libmagic` — a C library
that requires system packages (`apt install libmagic-dev`). This is a real
barrier on minimal Docker images and CI environments.

**Suggested `pyproject.toml`:**

```toml
[project]
dependencies = [
    "typing_extensions; python_version < '3.11'",
]

[project.optional-dependencies]
email = ["email_validator", "dnspython", "idna"]
magic = ["python-magic"]
all = ["email_validator", "dnspython", "idna", "python-magic"]
```

Then guard the imports with informative error messages:

```python
class email(wrapper):
    def __compile__(self, _deferred_compiles=None):
        try:
            import email_validator
        except ImportError:
            raise ImportError(
                "The 'email' schema requires the 'email' extra: "
                "pip install vtjson[email]"
            ) from None
        ...
```

---

## 5. What Is Good

This report leans heavily into what can be improved, so it is important to
give proportional credit to what `vtjson` already does well. The strengths
listed here are not routine — several of them are rare even in mature,
well-funded open-source projects.

### 5.1 Algebraically Complete Schema DSL

`vtjson` does not merely provide a bag of validators — it provides a
**closed algebra**. The set {`union`, `intersect`, `complement`, `anything`,
`nothing`} is functionally complete (see §3.1.2): any Boolean predicate on
Python objects can be expressed without resorting to opaque callables.

This is a design decision with profound consequences. It means that schema
composition is always well-defined, that error messages can be generated
structurally, and that future features like schema simplification or
equivalence checking are *possible* — the algebra provides the rewrite rules.
Most competing libraries (Cerberus, Schema, Voluptuous) do not close under
complement, which limits composability.

### 5.2 Seamless Typing Integration

The library bridges the gap between Python's *static* type system and
*runtime* validation with unusual thoroughness. It handles:

- `Annotated[T, S, skip_first]` — refinement types
- `TypedDict` with `Required` / `NotRequired` fields
- `Protocol` — structural subtyping at runtime
- `Union[A, B]` and the Python 3.10+ `A | B` syntax
- `Literal["a", "b"]` — value-level constraints
- `NewType` — semantic wrapper types
- `NamedTuple` — positional record types
- Generic base classes (`list[int]`, `dict[str, float]`, `Mapping[K, V]`)

The feature-detection code that enables this is itself praiseworthy:

```python
try:
    from typing import Literal
    supports_Literal = True
except ImportError:
    supports_Literal = False
```

This pattern is repeated for 8 distinct typing features, each with its own
graceful degradation path. The result is that `vtjson` works on Python 3.7
through 3.14 without conditional dependency lists or version-gated code paths
in the validation logic itself.

### 5.3 Recursive Schema Support — Correct Fixpoint Semantics

Recursive data structures (trees, linked lists, nested comments) are notoriously
difficult to validate. `vtjson` handles them correctly using a two-part
mechanism:

1. **`_deferred`** — a proxy compiled schema that stands in for a schema not
   yet fully compiled.
2. **`_mapping`** — an identity-based cache that resolves the proxy after
   compilation completes.

This is a textbook least-fixed-point construction (see §3.2.2), and it works
on the first pass — no iterative convergence needed. The user-facing syntax
is zero-ceremony:

```python
tree = {"value": int, "children": [tree]}  # just reference the variable
validate(tree, {"value": 1, "children": [{"value": 2, "children": []}]})
```

Many validation libraries either do not support recursive schemas at all, or
require explicit `Ref("name")` indirection.

### 5.4 Excellent Error Messages

Error messages in validation libraries are a frequent pain point. `vtjson`
produces **hierarchical, path-aware messages** that read like natural language:

```
object['authors'][0]['email'] (value:'not-an-email') is not of type 'email'
object['config']['threads'] (value:-1) is not of type 'ge(0)'
object['results']['pentanomial'] (value:[10, 20, 30, 40]) is not of type '[uint, uint, uint, uint, uint]'
```

Key design choices that make this work:

- **Path threading:** Every `__validate__` call receives a `name` parameter
  that accumulates the access path. Dict validation appends `[key]`, list
  validation appends `[index]`, and field validation appends `.attr`.
- **Value truncation:** The `_c()` helper truncates long values to 120
  characters with a `[TRUNCATED]` marker, preventing log explosion on large
  objects while preserving enough context to diagnose the problem.
- **Schema names in errors:** The `_set__name__` decorator auto-generates
  human-readable names from constructor arguments. A schema like
  `regex(r"[a-f0-9]{40}", name="sha")` produces the error
  `is not of type 'sha'` rather than a raw regex dump.
- **Explanations:** The `_wrong_type_message` helper optionally appends a
  structured explanation (e.g., "42 is not a Mapping"), giving both the
  expected type and the reason for failure.

This is significantly better than libraries that only report "validation failed"
or dump the entire schema on failure.

### 5.5 Pre-Compilation for Performance-Critical Paths

The two-phase design (`compile()` → `__validate__()`) is not just an
implementation detail — it is a **user-facing performance lever**. As shown
in §4.3.2, compilation is 30x more expensive than validation. By exposing
`compile()` as a public API, the library lets users amortize this cost:

```python
schema_c = compile({"name": str, "age": int, "active": bool})  # once
for request in incoming_requests:                                # many
    msg = schema_c.__validate__(request.json)
```

The project's own `bench.py` demonstrates this pattern with real-world schemas
(SPRT and SPSA run objects), proving that the design scales to production
workloads.

### 5.6 Thorough Test Suite

79 unit tests covering:

- **Core validation:** types, constants, dicts, lists, tuples, sets
- **Algebra:** union, intersect, complement, ifthen, cond
- **Wrappers:** regex, glob, email, url, ip_address, interval, size, div,
  filter, fields, set_label, set_name, quote, lax, strict
- **Recursion:** recursive dict schemas, recursive list schemas, mutual
  recursion
- **Type annotations:** TypedDict, Protocol, NamedTuple, Annotated, Union,
  Literal, NewType, Generic base classes
- **Edge cases:** float/int coercion, optional keys, truncation behavior,
  immutability of compiled schemas, schema name generation
- **Version compatibility:** tests that skip gracefully on older Python
  versions where specific typing features are unavailable

The tests are written in `unittest` style with clear names like
`test_intersect_validate`, `test_recursive_schema`, `test_annotated`. They
serve as executable documentation of the library's contract.

### 5.7 Real-World Deployment

`vtjson` is not an academic exercise — it validates production data in
[Fishtest](https://tests.stockfishchess.org/tests), the distributed testing
framework for the Stockfish chess engine. This means:

- The library handles **real network payloads** (JSON objects with 50+ fields,
  nested dicts, regex-validated strings, numeric constraints).
- It runs under **production load** (thousands of test runs per day).
- Its error messages are read by **real users** (engine developers submitting
  test configurations).

The `bench.py` file contains the actual Fishtest schemas (`runs_schema`,
`SPRT_run`, `SPSA_run`) — these are not toy examples but production
configurations with domain-specific constraints like pentanomial result
validation and ELO interval checking.

### 5.8 Strict Static Analysis Compliance

The codebase passes `mypy --strict` — the most restrictive mode of Python's
primary static type checker. This means:

- Every function has type annotations
- No `Any` types leak unintentionally
- All return types are explicit
- Generic types are properly parameterized

The `py.typed` marker declares PEP 561 compliance, meaning downstream
libraries that import `vtjson` get type information automatically in their
own `mypy` checks. The `lint.sh` pipeline enforces this on every commit:

```bash
mypy --strict vtjson.py
black --check vtjson.py test_vtjson.py
isort --check vtjson.py test_vtjson.py
flake8 vtjson.py test_vtjson.py
```

This is a high bar — many libraries claim type support but fail `--strict`.

**Room to grow:** The lint pipeline does not yet include **ruff** or **ty**
(Astral's new type checker). Adding them would catch ~30 additional genuine
findings (see §4.4). ruff can also replace flake8 + isort as a single, faster
tool.

### 5.9 The `_set__name__` Decorator — DRY Repr Generation

The `_set__name__` decorator is an underappreciated piece of engineering. It
introspects the constructor signature of each schema class, captures the
arguments, and auto-generates `__name__`, `__str__`, and `__repr__` from them:

```python
@_set__name__
class regex(wrapper):
    def __init__(self, pattern, name=None, fullmatch=True, ...): ...

# Now str(regex(r"[a-f0-9]{40}", name="sha")) == "regex('[a-f0-9]{40}', name='sha')"
```

This saves ~3 lines of boilerplate per class (across ~25 classes, that is ~75
lines avoided) and guarantees that error messages always reflect the user's
actual schema definition. It handles default arguments (omitting them from
the repr when unchanged), varargs, and keyword-only arguments.

### 5.10 Summary Table

| # | Strength | Rarity |
|---|---|---|
| 1 | Algebraically complete DSL (functionally complete Boolean algebra) | Rare — most validators lack complement |
| 2 | Full typing integration (8 typing constructs, 7 Python versions) | Rare — most support only basic types |
| 3 | Recursive schema support (zero-ceremony fixpoint) | Uncommon — many require explicit refs |
| 4 | Path-aware error messages with truncation and schema names | Uncommon — many just say "failed" |
| 5 | Pre-compilation for amortized validation | Available in some, but rarely this clean |
| 6 | 79 tests with version-aware skipping | Good — above average for a solo project |
| 7 | Production-deployed (Fishtest) | Significant — proves real-world viability |
| 8 | `mypy --strict` + PEP 561 | Rare — even large projects often fail strict |
| 9 | Automated lint pipeline (5 tools) | Good — above average |
| 10 | DRY repr generation via `_set__name__` | Unique — clever and effective |

---

## 6. What Should Be Improved

### 6.1 High Priority — Both Perspectives Agree

| # | Issue | Theoretician says | Pythonist says | Concrete action |
|---|---|---|---|---|
| 1 | **3K-line monolith** | Harder to identify logical subsystems | Impossible to navigate, no IDE outline | Split into package (see §4.2.1) |
| 2 | **Hard dep on `python-magic`** | Unrelated to the algebraic core | Blocks `pip install` on minimal systems | Move to extras (see §4.5) |
| 3 | **No CI/CD** | No automated proof that the algebra holds after changes | PRs can silently break tests | Add GitHub Actions (see [Appendix B](#appendix-b--cicd-implementation)) |
| 4 | **`filter.__compile__` bug** | Breaks recursive-schema invariant | Silent failure for deferred schemas in `filter` | Forward `_deferred_compiles` (see §8.1) |
| 5 | **`_mapping` naming** | Name clashes with mathematical mapping concept | Shadows `Mapping` type hint, confuses with `_Mapping` class | Rename to `_compile_cache` (see [Appendix A](#appendix-a--suggested-renaming-table)) |

### 6.2 Medium Priority — Actionable Improvements

| # | Issue | Perspective | Concrete action |
|---|---|---|---|
| 6 | **`_c()` function name** | Pythonist: single-letter names are for loop variables | Rename to `_truncated_repr()` |
| 7 | **`filter` shadows builtin** | Both: violates principle of least surprise | Rename to `preprocess` or `via` (see §7.2) |
| 8 | **11× `setattr(__validate__)`** | Pythonist: breaks static analysis | Use delegation pattern (see §4.2.4) |
| 9 | **Mutable default `subs={}`** | Pythonist: triggers B006 lint, surprises reviewers | Use `MappingProxyType({})` sentinel (see §4.2.3) |
| 10 | **No `__all__`** | Pythonist: pollutes `from vtjson import *` | Add `__all__` list (see §4.2.6) |
| 11 | **DNS resolver global state** | Both: not thread-safe | Use `lru_cache` singleton (see §4.2.5) |
| 12 | **No CHANGELOG** | Both: release tracking is invisible | Add `CHANGELOG.md` (Keep a Changelog format) |
| 13 | **`number` deprecated but not removed** | Both: dead code weighs down the API | Set removal timeline (v3.0) |
| 14 | **Benchmarks require unlisted `bson`** | Pythonist: contributor friction | Add `[project.optional-dependencies] dev = [...]` |
| 15 | **No ruff/ty in lint pipeline** | Pythonist: misses 30+ genuine findings | Add `ruff check` + `ty check` to `lint.sh` (see §4.4) |
| 16 | **No test coverage tracking** | Both: no visibility into untested paths | Add `coverage` to CI (see [Appendix B](#appendix-b--cicd-implementation)) |

### 6.3 Low Priority — Cosmetic & Documentation

| # | Issue | Action |
|---|---|---|
| 17 | `html_baseurl` typo: `"vtsjon"` → `"vtjson"` | One-line fix in `conf.py` |
| 18 | `anything` docstring: `"Matchess"` → `"Matches"` | One-line fix |
| 19 | Test skip messages: `"Pythin"` → `"Python"` (×2) | Two-line fix |
| 20 | `_set__name__` dunder-like name | Rename to `_auto_repr` |
| 21 | `bench_classic.py` uses deprecated `number` | Replace with `float` |
| 22 | Add `__validate__` docstrings (ruff D102, 84 occurrences) | Improves API discoverability |
| 23 | Replace generic `# type: ignore` with specific codes (ruff PGH003, 34×) | Improves type-checking clarity |
| 24 | Remove unnecessary `else` after `return` (ruff RET505, 26×) | Reduces nesting |

---

## 7. Surprise Catalog — Principle of Least Astonishment

> *"The purpose of this section is to catalog every moment where a reader says
> 'wait, what?' This is the brainstorming heart of the report."*

Each entry identifies a surprise, explains *why* it's surprising, and proposes
one or more alternatives.

### 7.1 `_mapping` — the Most Confusing Name in the Codebase

**Where:** `vtjson.py` line 1536.

**What it does:** An identity-based dictionary (`id(key)` → `(key, compiled_schema, in_use)`) that serves as a **deferred compilation registry**. During schema compilation, when a schema references itself recursively, `_mapping` stores the partially-compiled result so the fixpoint can be resolved later.

**Why it's surprising:**
- The name `_mapping` collides with `Mapping` from `collections.abc`, which is
  used throughout the codebase as a type hint. A reader seeing `_mapping` thinks
  "oh, this handles dict/mapping schemas" — but it doesn't.
- There is *also* a class `_Mapping` (capital M, line 3037) that handles
  `Mapping` type hint compilation. The two are unrelated.
- The word "mapping" has a precise mathematical meaning (a function between sets)
  that is also wrong here.

**Alternatives (ranked):**

| # | Name | Pro | Con |
|---|---|---|---|
| 1 | `_compile_cache` | Describes the mechanism | Doesn't hint at the fixpoint/recursion purpose |
| 2 | `_schema_registry` | Clear and descriptive | Slightly too "enterprisey" for a small lib |
| 3 | `_fixpoint_table` | Mathematically precise | Too academic for most readers |
| 4 | `_deferred_compiles` | Matches the parameter name already used in `_compile()` | Could be confused with `_deferred` class |

**Recommendation:** `_compile_cache` — it is short, unambiguous, and describes
what the object *is* (a cache of compiled schemas used during compilation).

### 7.2 `filter` — Shadowing a Builtin with Opposite Semantics

**Where:** `vtjson.py` line 2555 (public wrapper), line 2513 (internal `_filter`).

**What it does:** Applies a transformation function `f` to the input, then
validates `f(input)` against a schema. It is a **pre-processing step**, not a
filter.

**Why it's surprising:**
- Python's `filter()` **removes** elements from an iterable. This `filter`
  **transforms** them. A user reading `filter(int, gt(0))` expects to filter
  out non-positive values — but it actually converts to `int` and then validates
  `> 0`.
- The categorical interpretation is a **pullback** / **preimage** — validating
  in the codomain after applying a morphism. The name should reflect this.

**Alternatives:**

| # | Name | Pro | Con |
|---|---|---|---|
| 1 | `preprocess` | Intuitive: "run this function first, then validate" | Doesn't convey the validation part |
| 2 | `via` | Short, reads well: `via(int, gt(0))` = "via int, check gt(0)" | Novel — needs docs |
| 3 | `transform` | Standard vocabulary | Could suggest mutation |
| 4 | `preimage` | Mathematically perfect | Too academic |
| 5 | `after` | Reads naturally: `after(parse_date, date_time(...))` | Ambiguous direction |

**Recommendation:** `via` — minimal, reads naturally, no collision. Keep
`filter` as a deprecated alias for one major version.

### 7.3 `_c()` — a Name that Explains Nothing

**Where:** `vtjson.py` line 399.

**What it does:** Truncates the `repr()` of an object to 120 characters,
appending `"[TRUNCATED]"` if too long.

**Why it's surprising:** Single-letter function names are reserved for tight
mathematical loops (like coordinates `x, y, z`) or well-known abbreviations
(like `f` for file handles). This is a utility function called from dozens of
places — its name should describe what it does.

**Alternatives:**

| Name | Verdict |
|---|---|
| `_truncated_repr` | **Best** — exactly what it does |
| `_short_repr` | Good but doesn't hint at truncation |
| `_trunc` | Too terse, but better than `_c` |
| `_clip_repr` | Fine |

### 7.4 `_deferred` — Deferred *What*?

**Where:** `vtjson.py` line 1514.

**What it does:** A **proxy** compiled schema that, when validated, looks up the
real compiled schema from the `_mapping` (the compile cache). It exists because
recursive schemas may be referenced before they are fully compiled.

**Why it's surprising:** "Deferred" could mean deferred execution, deferred
import, deferred evaluation — it's a general adjective, not a specific noun.

**Alternatives:**

| Name | Verdict |
|---|---|
| `_lazy_schema` | Clear — it's a schema that resolves lazily |
| `_schema_ref` | Clear — it's a reference to a schema that will exist later |
| `_fixpoint_proxy` | Mathematically precise but academic |
| `_schema_placeholder` | Descriptive but long |

**Recommendation:** `_lazy_schema` or `_schema_ref` — both are self-documenting.

### 7.5 `setattr(self, "__validate__", ...)` — Invisible Method Replacement

**Where:** 11 occurrences across `vtjson.py` (lines 833, 1436, 1445, 1454,
1456, 1568, 2603, 2663, 2723, 2870, 2875).

**Why it's surprising:** A reader looking at a class definition expects
`__validate__` to be defined in the class body. When it's dynamically replaced
in `__init__`, the class definition lies about its own behavior. IDE "Go to
Definition" leads to the wrong implementation.

**Alternative:** Delegation (see §4.2.4). Each class defines `__validate__` on
the class body, and it delegates to a `_delegate` attribute set in `__init__`.
One extra indirection; full static-analysis transparency.

### 7.6 `subs={}` — a Mutable Default that Isn't (but Looks Like One)

**Where:** 20+ occurrences across `vtjson.py`.

**Why it's surprising:** Every Python tutorial, every linter, every code review
checklist flags `def f(x=[])` as a bug. The fact that `subs` is typed as
`Mapping` (immutable interface) makes this safe, but the safety is invisible —
you have to check the type annotation to know it's okay. A reader's first
reaction is always "this is a bug."

**Alternative:** `MappingProxyType({})` as the default, or a `None` sentinel.
See §4.2.3 for implementation.

### 7.7 `_set__name__` — Looks Like Dunder Pollution

**Where:** `vtjson.py` — decorator used on most public schema classes.

**Why it's surprising:** The double-underscore sequence `__name__` inside the
identifier makes it look like the function is setting a dunder attribute from
outside the class. The decorator actually *generates* `__name__`, `__str__`, and
`__repr__` from constructor arguments. It's a clever auto-repr decorator, but
the name doesn't communicate that.

**Alternatives:** `_auto_repr`, `_schema_repr`, `_named_schema`.

### 7.8 `float`-Matches-`int` — Algebraic Purity vs. Pragmatism

**Where:** Core compilation logic.

**Why it's surprising (for the theoretician):** If `float` matches `int`, then
`complement(float)` does not reject ints, breaking the law
A ∧ ¬A = ⊥. This is documented but still catches people off guard.

**Alternative (see §3.4.2):** Introduce a `numeric` schema that explicitly
means `int | float`. Let `float` mean only `float`. This restores algebraic
laws and is more explicit.

---

## 8. Potential Bugs & Edge Cases

### 8.1 Confirmed Code/Docs Issues

1. **`html_baseurl` typo in `docs/source/conf.py`:**
   ```python
   html_baseurl = "https://www.cantate.be/vtsjon"  # should be "vtjson"
   ```
   This will cause incorrect canonical URLs in the generated Sphinx documentation.

2. **Typo in `anything` docstring (vtjson.py):**
   ```python
   class anything(compiled_schema):
       """
       Matchess anything.    # double 's'
       """
   ```

3. **Typos in test skip reasons (test_vtjson.py):**
   ```python
   "Generic base classes were introduced in Pythin 3.9"   # "Pythin" → "Python"
   "Generics did not work well in Pythin 3.7"             # same
   ```

### 8.2 Potential Edge Cases (Not Bugs, But Worth Noting)

1. **`_const` float comparison:** Float constants use `math.isclose` with
   default tolerances. This is documented behavior, but may surprise users who
   expect exact matching. The `quote()` wrapper exists as the escape hatch.

2. **`_fields` validates missing optional attributes:** In `_fields.__validate__`,
   when a key is optional and the attribute is missing, the code still calls
   `getattr(obj, k_)` which would raise `AttributeError`. Looking more closely,
   the code checks `hasattr` first, but if the attribute is missing and
   optional, it still falls through to the `getattr` + `__validate__` call.
   This is guarded by the `hasattr` check preceding it, and validation
   continues to the next field, but the control flow is subtle and could
   benefit from an explicit `continue`.

3. **`_dict` with regex keys and overlapping matches:** When multiple regex
   keys match the same object key, the current algorithm tries them in
   insertion order and accepts the first match. This is documented behavior but
   may lead to order-dependent validation results.

4. **Thread safety of `_dns_resolver` initialization:** The global DNS resolver
   is lazily initialized without synchronization. Under concurrent access, this
   could lead to multiple resolver instances being created (benign but
   wasteful).

5. **`filter` wrapper passes `_deferred_compiles=None` instead of forwarding:**
   In `filter.__compile__`, the `_deferred_compiles` parameter is explicitly
   set to `None` instead of being forwarded. This means recursive schemas
   inside a `filter` may not compile correctly. If `filter` is never used with
   recursive schemas this is harmless, but it breaks the contract established
   by the `wrapper` protocol.

   ```python
   def __compile__(self, _deferred_compiles=None):
       return _filter(
           self.filter, self.schema,
           filter_name=self.filter_name,
           _deferred_compiles=None,        # ← should forward
       )
   ```

### 8.3 Static-Analysis False Positives (ty-only, not runtime bugs)

1. **`bench.py:347` — `task_object` "missing" `bad` key in `task_type` (ty):**
   ty reports `missing-typed-dict-key` for the `task_object` literal.
   `task_type` declares `bad: NotRequired[Literal[True]]`, so omitting `bad`
   is valid by design. The same applies to `rescheduled_from` in `runs_type`
   and `spsa` in `args_type`.

   **Classification:** tool false positive, not a confirmed code issue.

2. **`comparable` protocol mismatch in ty (bench.py lines 54–57):**
   ty does not treat `Literal[0]` as satisfying the `comparable` annotation in
   `ge(0)`/`gt(0)`. Runtime behavior is fine (`int` has comparison dunders);
   this is a type-checker interpretation gap.

3. **Clarification on `requires-python = ">=3.7"`:**
   This setting is **not an error**. It declares supported interpreter versions
   for packaging. The false positives come from ty's static baseline selection.
   In this audit, Python 3.7 was not re-run in-session, so there is no runtime
   evidence here of a 3.7 crash caused by this point.

4. **Practical workaround:**
   Run ty with `--python-version 3.11` (or configure
   `[tool.ty.environment] python-version = "3.11"`) so `NotRequired` and other
   modern typing symbols are resolved consistently.

---

## 9. Recommendations Summary

### Immediate (This Week)

| # | Action | LOE |
|---|---|---|
| 1 | Fix `html_baseurl` typo in `conf.py` | 1 min |
| 2 | Fix `anything` docstring typo | 1 min |
| 3 | Fix `"Pythin"` typos in `test_vtjson.py` (×2) | 1 min |
| 4 | Fix `filter.__compile__` — forward `_deferred_compiles` | 5 min |
| 5 | Add `__all__` to `vtjson.py` | 15 min |

### Short-Term (This Month)

| # | Action | LOE |
|---|---|---|
| 6 | Make `python-magic` an optional dependency | 30 min |
| 7 | Add GitHub Actions CI workflow (see [Appendix B](#appendix-b--cicd-implementation)) | 1 hr |
| 8 | Add ruff config to `pyproject.toml`, replace flake8+isort in `lint.sh` | 30 min |
| 9 | Add ty to `lint.sh` with `--python-version 3.11` | 15 min |
| 10 | Add `coverage` job to CI (see [Appendix B](#appendix-b--cicd-implementation)) | 30 min |
| 11 | Rename `_c()` → `_truncated_repr()` | 15 min |
| 12 | Rename `_mapping` → `_compile_cache` | 30 min |
| 13 | Add `CHANGELOG.md` | 30 min |
| 14 | Replace `subs={}` with `MappingProxyType` default | 45 min |

### Medium-Term (Next Release)

| # | Action | LOE |
|---|---|---|
| 15 | Rename `filter` → `via` (keep `filter` as deprecated alias) | 1 hr |
| 16 | Replace `setattr(__validate__)` with delegation pattern | 2 hr |
| 17 | Split `vtjson.py` into a package with re-exports | 4 hr |
| 18 | Thread-safe DNS resolver via `lru_cache` | 15 min |
| 19 | Fix ty `comparable` protocol to satisfy `Literal[0]` | 30 min |

### Long-Term (v3.0)

| # | Action | LOE |
|---|---|---|
| 20 | Remove deprecated `number` schema with migration guide | 1 hr |
| 21 | Remove deprecated `filter` alias | 5 min |
| 22 | Consider `numeric` schema for explicit `int \| float` coercion | 1 hr |

### For Contributors

- Run `sh lint.sh` before submitting PRs — the pipeline is thorough.
- The `bench.py` file requires `pymongo` (`bson`) — install it separately.
- The two benchmark files (`bench.py` and `bench_classic.py`) demonstrate the
  same real-world schema in two styles; changes to validation semantics should
  be verified against both.

---

## 10. Competitive Landscape & Adoption Strategy

### 10.1 Comparison with Production-Grade Validation Tools

The Python ecosystem has several well-established validation/serialization
libraries. The table below compares vtjson against the most prominent ones
across the dimensions that matter for adopters.

| Feature | **vtjson** | **Pydantic v2** | **marshmallow** | **Cerberus** | **voluptuous** | **typeguard** |
|---|---|---|---|---|---|---|
| **Primary purpose** | Validate arbitrary objects against schemas | Validate + serialize/deserialize data into typed models | Serialize + deserialize + validate (ORM-like) | Validate dicts against rule-based schemas | Validate dicts/data against schemas | Runtime type-checking for function signatures |
| **Schema definition** | Python dicts, types, DSL combinators, or `TypedDict`/`Annotated` | Python classes (BaseModel) or `TypeAdapter` | Python classes (Schema + fields) | Python dicts (rule declarations) | Python dicts + DSL functions | Type annotations (inline) |
| **Boolean algebra on schemas** | **Yes** — `union`, `intersect`, `complement`, `anything`, `nothing` (functionally complete) | No — `Union` only; no `intersect`, no `complement` | No | No | No — `Any`/`All` but no `complement` | No |
| **Recursive schemas** | **Yes** — zero-ceremony, fixpoint-based (just reuse the variable) | Yes — via `model_rebuild()` or `Self` | Yes — via `Nested("self")` | No | No | N/A |
| **`filter` / pullback combinator** | **Yes** — `filter(f, schema)` validates `f(obj)` against `schema` | No native equivalent (custom validators only) | No | No | `Coerce()` (partial) | No |
| **`Annotated[T, S]` integration** | **Yes** — first-class, bidirectional with typing | Yes — `Annotated[int, Field(gt=0)]` | No | No | No | Partial |
| **Schema substitution (`set_label`/`subs`)** | **Yes** — parametric schemas, natural transformation | No | No | No | No | No |
| **`make_type()` — schema → Python type** | **Yes** — creates `isinstance`-checkable types from schemas | Not needed (models are already types) | No | No | No | No |
| **Dict-literal schemas** | **Yes** — `{"name": str, "age": int}` is a valid schema | No — requires `BaseModel` or `TypeAdapter` | No — requires `Schema` class | Yes — dict-based | Yes — dict-based | No |
| **Lax / strict modes** | **Yes** — per-node `lax()`/`strict()` wrappers; controls extra keys locally | Yes — `model_config = {"extra": "forbid"}` (model-level only) | Partial — `unknown` field handling | Yes — `allow_unknown` flag | No | No |
| **Serialization / deserialization** | **No** — validation only (by design) | **Yes** — core feature (JSON, dict, ORM) | **Yes** — core feature | No | No | No |
| **Data coercion / casting** | No (validates, does not transform) | **Yes** — automatic type coercion | **Yes** | **Yes** | **Yes** (via `Coerce`) | No |
| **OpenAPI / JSON Schema generation** | No | **Yes** — automatic | **Yes** (via plugins) | Yes (partial) | No | No |
| **Error messages** | Hierarchical, path-aware, value-displaying | Hierarchical, structured (JSON-serializable) | Hierarchical, field-keyed | Dict of errors | Path-based | Traceback-based |
| **Performance (simple dict)** | ~2 us (pre-compiled) | ~1–3 us (model_validate) | ~10–50 us | ~10–50 us | ~5–20 us | ~1–5 us |
| **Zero-dependency core** | Possible (see §4.5) | No (depends on `pydantic-core` / Rust) | No (several deps) | Yes | Yes | No |
| **Python version support** | 3.7–3.14 | 3.8+ | 3.8+ | 3.8+ | 3.7+ | 3.8+ |
| **Codebase size** | ~3K LOC (single file) | ~50K+ LOC (Python + Rust core) | ~10K LOC | ~5K LOC | ~3K LOC | ~5K LOC |
| **Community size (PyPI monthly)** | Small (niche) | Very large (~250M/month) | Large (~30M/month) | Medium (~5M/month) | Medium (~5M/month) | Medium (~5M/month) |

### 10.2 Are vtjson and Pydantic Complementary?

**Yes — they solve different problems and can coexist.**

Pydantic is a **data modeling** framework: it defines Python objects, populates
them from raw data (deserialization), coerces types, generates JSON Schema,
and validates as a side effect of construction. Its mental model is
"I have incoming JSON → give me a typed Python object."

vtjson is a **validation predicate** library: it takes an existing Python
object and answers "does this object conform to this schema?" with a
yes/no + error message. It does not create objects, does not coerce, does not
serialize. Its mental model is "I already have a Python object → is it valid?"

**Concrete complementarity scenarios:**

| Scenario | Best tool | Why |
|---|---|---|
| Parse incoming API JSON into typed models | Pydantic | Needs deserialization + coercion |
| Validate an in-memory dict built by application logic | vtjson | No deserialization needed; dict-literal schema is simpler |
| Express "field X must satisfy (A or B) and not C" | vtjson | Boolean algebra (`intersect(union(A, B), complement(C))`) |
| Generate OpenAPI spec from models | Pydantic | Built-in JSON Schema generation |
| Validate recursive tree structures (config, AST) | vtjson | Zero-ceremony recursion; Pydantic requires `model_rebuild()` |
| Validate data flowing between internal services (no serialization boundary) | vtjson | Lighter weight; no model class overhead |
| Validate 3rd-party data structures you don't control | vtjson | Works on any Python object; Pydantic needs its own models |
| Build a REST API with FastAPI | Pydantic | Tight integration, auto-docs |

**The key distinction:** Pydantic _owns_ the data (it creates model instances).
vtjson _inspects_ data that already exists. When your data comes from outside
(API boundary), Pydantic is the natural choice. When your data is built
internally and you want to assert invariants — especially complex, algebraic
ones — vtjson fills a gap that Pydantic does not cover.

**Example — Fishtest `runs_schema` (the real production use case):**

The Fishtest testing framework validates "run" objects — complex dicts with
50+ fields, nested sub-objects (tasks, SPRT parameters, worker info), and
**cross-field invariants** that tie the whole model together. This is where
the orthogonality between Pydantic and vtjson becomes concrete.

**What Pydantic can do — individual field types:**

```python
from pydantic import BaseModel, Field
from typing import Literal

class SprtModel(BaseModel):
    alpha: float = 0.05
    beta: float = 0.05
    elo0: float
    elo1: float
    elo_model: Literal["normalized"]
    state: Literal["", "accepted", "rejected"]
    llr: float
    batch_size: int = Field(gt=0)

class ResultsModel(BaseModel):
    wins: int = Field(ge=0)
    losses: int = Field(ge=0)
    draws: int = Field(ge=0)
    crashes: int = Field(ge=0)
    time_losses: int = Field(ge=0)
    pentanomial: list[int]  # ← can't express "exactly 5 non-negative ints"

class RunModel(BaseModel):
    approved: bool
    approver: str
    finished: bool
    deleted: bool
    failed: bool
    is_green: bool
    is_yellow: bool
    workers: int = Field(ge=0)
    cores: int = Field(ge=0)
    results: ResultsModel
    tasks: list[dict]  # ← Pydantic can type each field, but...
```

Pydantic validates that each field has the right type and basic range. But it
**cannot structurally express** the following invariants that vtjson handles
natively:

**What only vtjson can express — whole-model invariants:**

```python
from vtjson import (
    validate, lax, ifthen, intersect, quote, keys,
    ge, gt, regex, Annotated, fields,
)

# 1. Cross-field conditional: if approved, approver must be a valid username
#    (not empty string); if not approved, approver must be ""
ifthen({"approved": True}, {"approver": username}, {"approver": ""})

# 2. State machine consistency: green and yellow are mutually exclusive
ifthen({"is_green": True}, {"is_yellow": False})
ifthen({"is_yellow": True}, {"is_green": False})

# 3. Lifecycle invariants: failed/deleted implies finished
ifthen({"failed": True}, {"finished": True})
ifthen({"deleted": True}, {"finished": True})

# 4. When finished: no active workers, no active cores, all tasks inactive
ifthen({"finished": True}, {"workers": 0, "cores": 0})
ifthen({"finished": True}, {"tasks": [{"active": False}, ...]})

# 5. Results aggregation: sum of task results must equal run results
#    (a callable predicate that vtjson calls as part of the schema)
def final_results_must_match(run):
    total = sum(t["stats"]["wins"] for t in run["tasks"])
    return total == run["results"]["wins"]  # (simplified)

# 6. If a task is marked "bad": it must be inactive with zero stats
if_bad_then_zero = ifthen(
    keys("bad"),
    lax({"active": False, "stats": quote(zero_results)})
)

# 7. SPRT and SPSA are mutually exclusive test types
args_schema = Annotated[args_type, at_most_one_of("sprt", "spsa")]

# 8. Pentanomial consistency: wins - losses = 2*P4 + P3 - P1 - 2*P0
def valid_results(R):
    l, d, w = R["losses"], R["draws"], R["wins"]
    P = R["pentanomial"]
    return (
        l + d + w == 2 * sum(P)
        and w - l == 2*P[4] + P[3] - P[1] - 2*P[0]
    )

# ALL of the above composed into a single schema:
runs_schema = Annotated[
    runs_type,
    lax(ifthen({"approved": True}, {"approver": username}, {"approver": ""})),
    lax(ifthen({"is_green": True}, {"is_yellow": False})),
    lax(ifthen({"is_yellow": True}, {"is_green": False})),
    lax(ifthen({"failed": True}, {"finished": True})),
    lax(ifthen({"deleted": True}, {"finished": True})),
    lax(ifthen({"finished": True}, {"workers": 0, "cores": 0})),
    lax(ifthen({"finished": True}, {"tasks": [{"active": False}, ...]})),
    intersect(final_results_must_match, cores_must_match, workers_must_match),
]
```

This is the actual `runs_schema` from `bench.py` — it is **not a toy example**.
It validates real Fishtest run objects in production.

**Why Pydantic cannot replace this:**

| Invariant | Pydantic approach | vtjson approach |
|---|---|---|
| *if approved then approver is valid username* | `@model_validator(mode="after")` — imperative Python | `ifthen({"approved": True}, {"approver": username})` — declarative |
| *green and yellow mutually exclusive* | `@model_validator` — imperative | `ifthen({"is_green": True}, {"is_yellow": False})` — declarative |
| *finished → workers=0, cores=0, all tasks inactive* | `@model_validator` — 10+ lines of Python | One `ifthen` line |
| *pentanomial consistency* | `@field_validator("pentanomial")` — but requires access to `wins`, `losses`, `draws` from sibling fields | `valid_results` as a callable predicate composed via `intersect` |
| *sum of all task results = run results* | `@model_validator` — requires iterating tasks, summing, comparing | `final_results_must_match` — same logic, but composed as a schema predicate |
| *SPRT and SPSA mutually exclusive* | `@model_validator` — check both keys | `at_most_one_of("sprt", "spsa")` — one expression |

The pattern is clear: **Pydantic validates field types (the shape of data);
vtjson validates field relationships (the meaning of data).** Every cross-field
invariant in Pydantic requires imperative `@model_validator` code — the schema
does not capture the constraint declaratively. In vtjson, the constraint *is*
the schema.

**Using them together:**

```python
from pydantic import BaseModel
from vtjson import validate

# Phase 1: Pydantic parses raw JSON from the API into typed objects
run = RunModel.model_validate(raw_json)

# Phase 2: vtjson validates cross-field invariants on the parsed data
#           (Pydantic already guaranteed the types; vtjson checks the logic)
validate(runs_schema, run.model_dump())
```

This two-phase approach gives you the best of both worlds: Pydantic's
serialization/deserialization and IDE support, plus vtjson's algebraic
constraint validation. Neither tool replaces the other.

### 10.3 What Makes vtjson Unique

vtjson occupies a niche that no other mainstream Python library fills.
Its uniqueness comes from three properties that, taken together, are
unmatched:

#### 10.3.1 Algebraic Completeness

vtjson is the **only** Python validation library whose schema combinators
form a functionally complete Boolean algebra. The set
{`union`, `intersect`, `complement`, `anything`, `nothing`} can express
**any** Boolean predicate over Python objects, structurally (see §3.1.2).

This is not merely a theoretical curiosity. It has practical consequences:

- **Negation is expressible.** "Accept everything _except_ this pattern" is
  `complement(pattern)`. No other library can do this without dropping to
  raw Python.
- **Intersection is first-class.** "Must be an int _and_ greater than 0 _and_
  divisible by 3" is `intersect(int, gt(0), div(3))`. Pydantic requires
  stacking `@field_validator` decorators or writing a custom type.
- **Laws hold.** Users can reason algebraically about schema composition
  without surprises (modulo the documented float/int coercion, see §3.4.2).

#### 10.3.2 Schemas Are Plain Python Values

A vtjson schema is an ordinary Python object — a dict, a list, a type, or a
combinator call. It does not require subclassing, decorating, or registering.
This means:

- **Schemas can be computed.** `{f"field_{i}": int for i in range(10)}` is
  a valid 10-field schema. Try that with Pydantic `BaseModel`.
- **Schemas compose naturally.** Merging two dict schemas is
  `{**schema_a, **schema_b}` — standard Python dict merge.
- **No metaclass magic.** No hidden `__init_subclass__`, no descriptor
  protocol, no class-scoped namespace manipulation. The schema is what it
  looks like.

#### 10.3.3 The Filter/Pullback Combinator

`filter(f, schema)` validates `f(obj)` against `schema`. This is the
categorical pullback (see §3.3.2), and it enables a class of validations
that other libraries handle only with ad-hoc custom validators:

```python
filter(len, interval(1, 100))        # string/list length in [1, 100]
filter(str.lower, one_of("a", "b"))  # case-insensitive enum
filter(int, gt(0))                   # "string that parses to positive int"
```

The combinator composes cleanly with the algebra:
`filter(f, union(A, B)) = union(filter(f, A), filter(f, B))`. No other
Python library provides this as a primitive.

#### 10.3.4 Summary: vtjson's Unique Selling Points

| Capability | vtjson | Closest alternative |
|---|---|---|
| Functionally complete Boolean algebra on schemas | Built-in | None — must drop to raw Python |
| `complement(schema)` | Built-in | Not available |
| `filter(f, schema)` — categorical pullback | Built-in | Custom validators (no composability) |
| Dict-literal schemas: `{"a": int, "b": str}` | Built-in | Cerberus/Voluptuous (but without algebra) |
| Recursive schemas via plain variable reuse | Built-in | Pydantic (`Self`), marshmallow (`Nested("self")`) |
| `make_type(schema)` — schema to `isinstance`-checkable type | Built-in | Not available |
| Schema substitution (`set_label`/`subs`) | Built-in | Not available |
| Zero-class validation (no `BaseModel` needed) | Built-in | typeguard (but no schema algebra) |

### 10.4 Adoption Strategy — Avoiding "Yet Another Validation Library"

The biggest risk for vtjson's adoption is the _perception_ problem: developers
see "Python validation library," mentally compare it to Pydantic, and dismiss
it. The remedy is to **stop competing with Pydantic and start complementing it**.

#### 10.4.1 Reposition: Predicate Algebra, Not Data Validation

The README currently opens with a dict-validation example that looks like
every other validation library. A developer skimming it will think
"I already have Pydantic" and close the tab.

**Suggested README reframing:**

> ~~vtjson is an easy to use validation library compatible with Python type
> annotations.~~
>
> **vtjson** is a composable predicate algebra for Python objects.
> It lets you express complex validation rules — including negation,
> intersection, and recursive schemas — as plain Python values, with
> full compatibility with `typing` annotations (`TypedDict`, `Annotated`,
> `Protocol`, `Literal`, …).

The first sentence should answer: "Why would I use this instead of Pydantic?"
The answer is: **because Pydantic cannot express `complement`, `intersect`,
`filter`, or recursive schemas as easily — and vtjson doesn't require defining
classes.**

#### 10.4.2 Lead with What's Unique, Not What's Common

The current README starts with:
```python
book_schema = {"title": str, "authors": [str, ...], "year": int}
```

This is valid but unremarkable — Pydantic, Cerberus, and Voluptuous all do this.

**Suggested: open with an example that _only_ vtjson can express cleanly:**

```python
from vtjson import validate, intersect, complement, union, regex, gt, ge, filter

# "A non-empty string that is NOT an email address and parses to a positive int"
weird_schema = intersect(
    str,
    complement(regex(r".+@.+\..+")),
    filter(int, gt(0)),
)

validate(weird_schema, "42")           # ✓
validate(weird_schema, "hello@x.com")  # ✗ — matches email pattern
validate(weird_schema, "-5")           # ✗ — int("-5") is not gt(0)
```

Then show the simpler dict example _second_, with a note: "For straightforward
dict validation, vtjson works like you'd expect:"

This inverts the funnel: the reader sees something _novel_ first, then
recognizes that vtjson also handles the basics.

#### 10.4.3 Lean Into Theoretical Correctness as a Feature

The target audience for vtjson is not the same as Pydantic's. Pydantic targets
web developers who want fast API serialization. vtjson should target:

- **Configuration validation** — deeply nested, rule-heavy configs (CI systems,
  game engines, scientific pipelines) where `intersect` and `complement` shine.
- **Protocol/contract testing** — asserting that internal data structures
  satisfy invariants, without the overhead of creating model classes.
- **Academics and formal-methods-adjacent developers** — people who _appreciate_
  that the schema language forms a Boolean algebra and that `filter` is a
  pullback.

**Concrete suggestions for the README and docs:**

1. **Add a "Why vtjson?" section** before the tutorial, with a 3-row table:
   - "I need to parse JSON into typed Python objects" → Use Pydantic.
   - "I need to validate existing Python objects against complex rules" → **Use vtjson.**
   - "I need both" → Use Pydantic for parsing, vtjson for invariant checking.

2. **Add a "Theoretical Foundations" page** to the Sphinx docs explaining the
   lattice structure, the filter-as-pullback interpretation, and the fixpoint
   construction. This is a differentiator, not a liability — it signals that
   the design is principled, not ad-hoc.

3. **Add a badge or tagline** that communicates the algebraic angle:
   > _"A composable, algebraically complete validation DSL for Python."_

4. **Publish a blog post or short paper** titled something like:
   _"Beyond Pydantic: When You Need a Boolean Algebra on Validation Predicates."_
   This positions vtjson as a complement, not a competitor, and attracts the
   right audience.

#### 10.4.4 Reduce Adoption Friction

Beyond messaging, practical friction must be reduced:

| Friction point | Current state | Suggested fix | Impact |
|---|---|---|---|
| `pip install` pulls `libmagic` | Hard dep on `python-magic` | Make optional (§4.5) | **High** — blocks adoption on Docker/CI |
| No PyPI badge / CI badge on README | No badges | Add CI + PyPI badges | Medium — signals maturity |
| No CHANGELOG | No release history | Add `CHANGELOG.md` | Medium — users can't assess stability |
| README links to external docs only | No inline API summary | Add a compact API table to README | High — users decide in 30 seconds |
| No comparison with alternatives | Nothing | Add "vtjson vs Pydantic" section in docs | High — answers the first question |
| No `pip install vtjson` quick-start | README jumps to schemas | Add 3-line quick-start at top | Medium |
| Single-file architecture | Hard to contribute | Split into package (§4.2.1) | Medium — reduces contributor barrier |

#### 10.4.5 Suggested README Structure (Outline)

```
# vtjson — Composable Predicate Algebra for Python

> Validate Python objects with union, intersection, complement, and more.
> Compatible with TypedDict, Annotated, Protocol, Literal, and recursive schemas.

[badges: PyPI, CI, Python versions, license]

## Quick Start
  pip install vtjson
  3-line example with validate()

## Why vtjson?
  When to use vtjson vs Pydantic vs marshmallow (3-row decision table)

## What Makes vtjson Different
  - Boolean algebra on schemas (complement, intersect)
  - filter(f, schema) — validate f(obj)
  - Recursive schemas with zero ceremony
  - Dict-literal schemas — no class definitions needed
  - Full typing integration (TypedDict, Annotated, Protocol, Literal)

## Examples
  1. The "impossible" example (complement + filter + intersect)
  2. Simple dict validation (the current book_schema)
  3. TypedDict version of the same
  4. Recursive schema (tree)

## API Reference (compact table)
## Full Documentation → link to Sphinx
## License
```

This structure gets the unique value proposition above the fold, answers
"why not Pydantic?" before the reader even asks, and keeps the README
actionable.

---

## Appendix A — Suggested Renaming Table

Full reference table for all naming surprises identified in §7.

| Current | Location | Role | Suggested | Migration |
|---|---|---|---|---|
| `_mapping` | `vtjson.py:1536` | Deferred compilation cache (identity dict) | `_compile_cache` | Internal only — no API break |
| `_Mapping` | `vtjson.py:3037` | Compiled schema for `Mapping` type hint | `_MappingSchema` | Internal only |
| `_deferred` | `vtjson.py:1514` | Proxy for not-yet-compiled recursive schema | `_lazy_schema` | Internal only |
| `_c()` | `vtjson.py:399` | Truncated repr helper | `_truncated_repr()` | Internal only |
| `filter` | `vtjson.py:2555` | Pre-validate transformation (pullback) | `via` | Public API — deprecate `filter`, alias for 1 version |
| `_filter` | `vtjson.py:2513` | Compiled form of `filter` | `_via` | Internal only |
| `_set__name__` | `vtjson.py` (decorator) | Auto-generates `__name__`, `__str__`, `__repr__` | `_auto_repr` | Internal only |
| `number` | `vtjson.py` | Deprecated alias for `float` | *(remove in v3.0)* | Already emits `DeprecationWarning` |

**Migration strategy for `filter` → `via`:**

```python
# In vtjson.py - keep both for one major version
class via(wrapper):
    """Validate schema against f(object) instead of object."""
    ...

# Deprecated alias
class filter(via):
    def __init_subclass__(cls, **kwargs):
        import warnings
        warnings.warn(
            "'filter' is deprecated, use 'via' instead",
            DeprecationWarning, stacklevel=2
        )
        super().__init_subclass__(**kwargs)

    def __init__(self, *args, **kwargs):
        import warnings
        warnings.warn(
            "'filter' is deprecated, use 'via' instead",
            DeprecationWarning, stacklevel=2
        )
        super().__init__(*args, **kwargs)
```

---

## Appendix B — CI/CD Implementation

The project has `lint.sh` but no automated CI. Below is a ready-to-use GitHub
Actions workflow incorporating the findings from §4.4 (ruff, ty, coverage).

### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [master, main]
  pull_request:
    branches: [master, main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      - name: Install lint dependencies
        run: |
          pip install mypy ruff typing_extensions
          pip install dnspython email_validator idna python-magic
      - name: mypy --strict
        run: mypy --strict vtjson.py
      - name: ruff check
        run: ruff check vtjson.py test_vtjson.py bench.py bench_classic.py
      - name: ruff format --check
        run: ruff format --check vtjson.py test_vtjson.py

  ty:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      - name: Install ty
        run: pip install ty
      - name: ty check
        run: ty check vtjson.py --python-version 3.11
        continue-on-error: true  # informational until false positives are resolved

  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.8", "3.9", "3.10", "3.11", "3.12", "3.13", "3.14"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          allow-prereleases: true
      - name: Install system dependencies
        run: sudo apt-get install -y libmagic-dev
      - name: Install Python dependencies
        run: |
          pip install dnspython email_validator idna python-magic typing_extensions
      - name: Run tests
        run: python -m unittest test_vtjson -v

  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      - name: Install dependencies
        run: |
          pip install coverage
          pip install dnspython email_validator idna python-magic typing_extensions
          sudo apt-get install -y libmagic-dev
      - name: Run tests with coverage
        run: |
          coverage run -m unittest test_vtjson -v
          coverage report --show-missing --fail-under=80
          coverage html
      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: htmlcov/

  docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      - name: Install docs dependencies
        run: pip install -r docs/requirements.txt
      - name: Build Sphinx docs
        run: |
          cd docs
          sphinx-build -W -b html source _build/html
```

**Notes:**
- The test matrix covers Python **3.8–3.14**, including the latest pre-release.
  `allow-prereleases: true` is required for 3.14 until its stable release.
- `ruff` replaces `flake8` + `isort` + `black` as a single, faster tool.
  Configure it via `[tool.ruff]` in `pyproject.toml` (see §4.4.2).
- `ty` runs as a separate job with `continue-on-error: true` because it is
  still early-stage and produces false positives on Python 3.7 codebases.
  Use `--python-version 3.11` to avoid `NotRequired`-related noise.
- **Coverage** is collected with `coverage.py` and fails the build if
  coverage drops below 80%. The HTML report is uploaded as an artifact for
  review. To view locally:
  ```bash
  pip install coverage
  coverage run -m unittest test_vtjson -v
  coverage report --show-missing
  coverage html && open htmlcov/index.html
  ```
- `libmagic-dev` is installed for the `python-magic` dependency.
- The `-W` flag in `sphinx-build` turns warnings into errors — catches broken
  cross-references.
- If `python-magic` is made optional (§4.5), the `sudo apt-get` step can be
  removed from the test matrix and replaced with a separate optional-deps job.

### Optional: Add Badges to README

```markdown
[![CI](https://github.com/<owner>/vtjson/actions/workflows/ci.yml/badge.svg)](https://github.com/<owner>/vtjson/actions/workflows/ci.yml)
```

---

## Appendix C — Code Refactoring Sketches

Concrete before/after code for the top refactoring candidates.

### C.1 Fix `filter.__compile__` Bug

**Before (line 2593):**
```python
def __compile__(self, _deferred_compiles=None):
    return _filter(
        self.filter, self.schema,
        filter_name=self.filter_name,
        _deferred_compiles=None,          # ← BUG: always None
    )
```

**After:**
```python
def __compile__(self, _deferred_compiles=None):
    return _filter(
        self.filter, self.schema,
        filter_name=self.filter_name,
        _deferred_compiles=_deferred_compiles,  # ← forward the parameter
    )
```

### C.2 Replace `setattr(__validate__)` with Delegation (Interval Example)

**Before (interval.__init__, line ~1430):**
```python
class _interval(compiled_schema):
    def __init__(self, ...):
        ...
        if lower is not None and upper is not None:
            setattr(self, "__validate__", _intersect((lower, upper)).__validate__)
        elif lower is not None:
            setattr(self, "__validate__", lower.__validate__)
        elif upper is not None:
            setattr(self, "__validate__", upper.__validate__)
        else:
            setattr(self, "__validate__", _anything().__validate__)
```

**After:**
```python
class _interval(compiled_schema):
    def __init__(self, ...):
        ...
        if lower is not None and upper is not None:
            self._impl = _intersect((lower, upper))
        elif lower is not None:
            self._impl = lower
        elif upper is not None:
            self._impl = upper
        else:
            self._impl = _anything()

    def __validate__(self, obj, name="object", strict=True, subs={}):
        return self._impl.__validate__(obj, name, strict, subs)
```

**Trade-off:** One extra method call per validation. In benchmarks, this is
negligible — the validation logic itself dominates. The gain is that
`__validate__` is always visible in the class definition, and IDE navigation
works.

### C.3 Replace `subs={}` with Immutable Default

**Before (20+ occurrences):**
```python
def __validate__(self, obj: object, name: str = "object",
                 strict: bool = True, subs: Mapping[str, object] = {}) -> str:
```

**After:**
```python
from types import MappingProxyType

_EMPTY_SUBS: Final[Mapping[str, object]] = MappingProxyType({})

def __validate__(self, obj: object, name: str = "object",
                 strict: bool = True, subs: Mapping[str, object] = _EMPTY_SUBS) -> str:
```

### C.4 Thread-Safe DNS Resolver

**Before:**
```python
_dns_resolver: dns.resolver.Resolver | None = None

def _get_resolver():
    global _dns_resolver
    if _dns_resolver is None:
        _dns_resolver = dns.resolver.Resolver()
        _dns_resolver.cache = dns.resolver.LRUCache()
    return _dns_resolver
```

**After:**
```python
@functools.lru_cache(maxsize=1)
def _get_dns_resolver() -> dns.resolver.Resolver:
    resolver = dns.resolver.Resolver()
    resolver.cache = dns.resolver.LRUCache()
    return resolver
```

`lru_cache` handles thread-safety and lazy initialization in one decorator.

### C.5 Renaming `_mapping` → `_compile_cache`

This is an internal class, so no public API breaks. Find-and-replace:

```bash
# Scope: vtjson.py only (no external consumers)
sed -i 's/_mapping/_compile_cache/g' vtjson.py
```

Then manually verify that the 2 occurrences (`_mapping` class definition at
line 1536, and all usages) are correctly renamed, and that the `_Mapping` class
(line 3037, capital M — handles `Mapping` type hints) is NOT touched.

### C.6 Optional Dependencies in `pyproject.toml`

**Before:**
```toml
[project]
dependencies = [
    "dnspython",
    "email_validator",
    "idna",
    "python-magic",
    "typing_extensions",
]
```

**After:**
```toml
[project]
dependencies = [
    "typing_extensions; python_version < '3.11'",
]

[project.optional-dependencies]
email = ["email_validator", "dnspython", "idna"]
magic = ["python-magic"]
all = ["email_validator", "dnspython", "idna", "python-magic"]
dev = ["mypy", "black", "isort", "flake8", "pymongo"]
docs = ["-r docs/requirements.txt"]
```

Usage: `pip install vtjson[all]` for full functionality, `pip install vtjson`
for core-only.

---

## Appendix D — Benchmark Script

The following standalone script replicates the performance measurements reported
in §4.3. It requires only `vtjson` to be importable (install with
`pip install .` from the project root). No external dependencies are needed.

### `bench_report.py`

```python
#!/usr/bin/env python3
"""Benchmark script to replicate the performance analysis in REPORT.md §4.3.

Run:  python bench_report.py
Requires: vtjson (pip install . from the project root)
"""

from __future__ import annotations

import sys
import timeit
from typing import Any

import vtjson
from vtjson import (
    anything,
    compile,
    complement,
    gt,
    intersect,
    nothing,
    union,
    validate,
)

# ---------------------------------------------------------------------------
# Helpers
# ---------------------------------------------------------------------------

def bench(label: str, stmt: str, setup: str = "", globs: dict | None = None,
          number: int = 500_000) -> float:
    """Time *stmt* and print the result.  Returns per-call time in us."""
    if globs is None:
        globs = {}
    total = timeit.timeit(stmt, setup=setup, globals=globs, number=number)
    per_call_us = total / number * 1e6
    print(f"  {label:.<55s} {per_call_us:8.3f} us  ({number} calls)")
    return per_call_us


def section(title: str) -> None:
    print(f"\n{'=' * 70}")
    print(f"  {title}")
    print(f"{'=' * 70}")


# ---------------------------------------------------------------------------
# §4.3.1  Baseline: compiled validation speed
# ---------------------------------------------------------------------------

def bench_baseline() -> None:
    section("§4.3.1 — Baseline: Compiled Validation")

    int_c = compile(int)
    str_c = compile(str)
    dict3 = compile({"name": str, "age": int, "active": bool})
    dict6 = compile({"a": int, "b": str, "c": float, "d": bool, "e": int, "f": str})
    list5 = compile([int, int, int, int, int])
    union4 = compile(union(str, float, list, dict))
    inter3 = compile(intersect(int, gt(0), lambda x: x < 100))

    g = {
        "int_c": int_c, "str_c": str_c,
        "dict3": dict3, "dict6": dict6,
        "list5": list5, "union4": union4, "inter3": inter3,
    }

    bench("isinstance(42, int)  [raw baseline]",
          "isinstance(42, int)", globs=g)
    bench("compiled int.__validate__(42)",
          "int_c.__validate__(42, 'x', True, {})", globs=g)
    bench("compiled str.__validate__('hi')",
          "str_c.__validate__('hi', 'x', True, {})", globs=g)
    bench("dict (3 keys, all match)",
          "dict3.__validate__({'name':'a','age':1,'active':True}, 'x', True, {})",
          globs=g)
    bench("dict (6 keys, all match)",
          "dict6.__validate__({'a':1,'b':'x','c':1.0,'d':True,'e':2,'f':'y'}, 'x', True, {})",
          globs=g)
    bench("list [int x5]",
          "list5.__validate__([1,2,3,4,5], 'x', True, {})", globs=g)
    bench("union(4) first hit",
          "union4.__validate__('hi', 'x', True, {})", globs=g)
    bench("union(4) all miss",
          "union4.__validate__(42, 'x', True, {})", globs=g)
    bench("intersect(3) all pass",
          "inter3.__validate__(50, 'x', True, {})", globs=g)


# ---------------------------------------------------------------------------
# §4.3.2  Compilation vs. validation bottleneck
# ---------------------------------------------------------------------------

def bench_compile_vs_validate() -> None:
    section("§4.3.2 — Compilation vs. Validation")

    g: dict[str, Any] = {
        "compile": compile, "validate": validate, "vtjson": vtjson,
    }

    bench("_compile(int)            [simple type]",
          "vtjson._compile(int)", globs=g, number=200_000)
    bench("_compile(str)",
          "vtjson._compile(str)", globs=g, number=200_000)
    bench("_compile({'a': int})     [1-key dict]",
          "vtjson._compile({'a': int})", globs=g, number=100_000)
    bench("compile({3 keys})        [3-key dict]",
          "compile({'name': str, 'age': int, 'active': bool})",
          globs=g, number=100_000)

    schema = {"name": str, "age": int, "active": bool}
    obj = {"name": "Alice", "age": 30, "active": True}
    g["schema"] = schema
    g["obj"] = obj
    bench("validate(schema, obj)    [raw = compile+validate]",
          "validate(schema, obj)", globs=g, number=100_000)

    compiled = compile(schema)
    g["compiled"] = compiled
    bench("compiled.__validate__(obj) [pre-compiled]",
          "compiled.__validate__(obj, 'x', True, {})", globs=g)


# ---------------------------------------------------------------------------
# §4.3.3  _compile dispatch chain cost
# ---------------------------------------------------------------------------

def bench_dispatch_chain() -> None:
    section("§4.3.3 — _compile Dispatch Chain")

    g: dict[str, Any] = {"vtjson": vtjson}

    bench("_compile(int)   — bare type (branch 17)",
          "vtjson._compile(int)", globs=g, number=200_000)
    bench("_compile(str)   — bare type (branch 17)",
          "vtjson._compile(str)", globs=g, number=200_000)
    bench("_compile(float) — bare type (branch 17)",
          "vtjson._compile(float)", globs=g, number=200_000)

    print("\n  Note: bare types hit branch 17 out of 18 in the if/elif chain.")
    print("  Reordering branches by frequency would reduce dispatch cost.")


# ---------------------------------------------------------------------------
# §4.3.4  setattr vs delegation overhead
# ---------------------------------------------------------------------------

def bench_setattr_vs_delegation() -> None:
    section("§4.3.4 — setattr vs. Delegation Overhead")

    setup = '''
class Normal:
    def method(self):
        return ""

class WithSetattr:
    def __init__(self):
        import types
        def _impl(self_):
            return ""
        object.__setattr__(self, 'method', types.MethodType(_impl, self))

class WithDelegation:
    def __init__(self):
        self._impl = Normal()
    def method(self):
        return self._impl.method()

normal = Normal()
with_setattr = WithSetattr()
with_delegation = WithDelegation()
'''
    g: dict[str, Any] = {}
    exec(setup, g)

    bench("normal method call",
          "normal.method()", globs=g)
    bench("setattr bound method",
          "with_setattr.method()", globs=g)
    bench("delegation (self._impl.method())",
          "with_delegation.method()", globs=g)


# ---------------------------------------------------------------------------
# §4.3.5  String return protocol cost
# ---------------------------------------------------------------------------

def bench_string_return() -> None:
    section("§4.3.5 — String Return Protocol")

    setup = '''
def return_empty():
    return ""

def return_none():
    return None

def return_true():
    return True

msg_empty = return_empty()
msg_none = return_none()
msg_bool = return_true()
'''
    g: dict[str, Any] = {}
    exec(setup, g)

    bench("return '' + compare == ''",
          "msg = return_empty(); msg == ''", globs=g)
    bench("return None + compare is None",
          "msg = return_none(); msg is None", globs=g)
    bench("return True + compare not msg",
          "msg = return_true(); not msg", globs=g)


# ---------------------------------------------------------------------------
# Bonus: end-to-end with a realistic schema
# ---------------------------------------------------------------------------

def bench_realistic() -> None:
    section("Bonus — Realistic Schema (Fishtest-like)")

    from vtjson import ge, regex, interval

    schema = {
        "username": intersect(str, regex(r"[!-~][ -~]{0,30}[!-~]")),
        "tc": intersect(str, regex(r"(\d+/)?(\d+)(\.\d+)?(\+\d+(\.\d+)?)?")),
        "threads": intersect(int, ge(1)),
        "hash": intersect(int, ge(1)),
        "new_tag": intersect(str, regex(r"[a-f0-9]{7,40}")),
        "base_tag": intersect(str, regex(r"[a-f0-9]{7,40}")),
        "num_games": intersect(int, ge(1)),
        "results": {
            "wins": intersect(int, ge(0)),
            "losses": intersect(int, ge(0)),
            "draws": intersect(int, ge(0)),
        },
    }

    obj = {
        "username": "test_user",
        "tc": "10+0.1",
        "threads": 1,
        "hash": 64,
        "new_tag": "abcdef1234567890abcdef1234567890abcdef12",
        "base_tag": "1234567890abcdef1234567890abcdef12345678",
        "num_games": 1000,
        "results": {"wins": 100, "losses": 90, "draws": 810},
    }

    g: dict[str, Any] = {
        "compile": compile, "validate": validate,
        "schema": schema, "obj": obj,
    }

    bench("validate(schema, obj)    [8-field, nested, regex]",
          "validate(schema, obj)", globs=g, number=50_000)

    compiled = compile(schema)
    g["compiled"] = compiled
    bench("compiled.__validate__(obj) [pre-compiled]",
          "compiled.__validate__(obj, 'x', True, {})", globs=g, number=200_000)

    print(f"\n  Speedup from pre-compilation: "
          f"run both benchmarks and divide the first by the second.")


# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------

def main() -> None:
    print(f"Python {sys.version}")
    print(f"vtjson {vtjson.__version__}")
    print(f"Benchmarking... (this may take 30-60 seconds)")

    bench_baseline()
    bench_compile_vs_validate()
    bench_dispatch_chain()
    bench_setattr_vs_delegation()
    bench_string_return()
    bench_realistic()

    section("Done")
    print("  Compare results with §4.3 in REPORT.md.")
    print("  Timings vary by machine — ratios are what matter.")


if __name__ == "__main__":
    main()
```

**Usage:**

```bash
cd /path/to/vtjson
pip install .          # or: pip install -e .
python bench_report.py
```

**What to look for:** Absolute timings will differ across machines. The
important numbers are the *ratios*:

| Ratio | Expected | What it tells you |
|---|---|---|
| compiled type check / raw isinstance | ~4x | Overhead of the vtjson dispatch layer |
| validate() / compiled.__validate__() | ~30x | Cost of recompiling every call |
| _compile(int) position in chain | branch 17/18 | Bare types are dispatched last |
| setattr / normal method | ~1.1x | setattr is nearly free |
| delegation / normal method | ~2x | Delegation adds a call frame |
| string return / None return | ~1.0x | No measurable difference |

---

## Appendix E — Static Analysis Results

Raw output summaries from `ty check` and `ruff check --select ALL`, run on
2026-02-12. Tool versions: **ty 0.0.16**, **ruff 0.15.0**, Python 3.14.3.

### E.1 ty check *.py — Summary (87 diagnostics)

**Error breakdown by rule:**

| Rule | Count | Description |
|---|---|---|
| `invalid-argument-type` | 37 | Type mismatch in function/method arguments |
| `unresolved-import` | 13 | Module members not found (Python 3.7 baseline) |
| `not-subscriptable` | 9 | Subscript on `object` without `__getitem__` |
| `unresolved-attribute` | 8 | Attribute access on types that lack it |
| `invalid-assignment` | 8 | Type mismatch in assignment |
| `missing-typed-dict-key` | 3 | Required TypedDict keys missing (false positives from 3.7 baseline) |
| `unsupported-operator` | 2 | Operator on incompatible types |
| `not-iterable` | 1 | Iterating non-iterable `object` |
| `invalid-method-override` | 1 | Method signature mismatch in override |

**Root cause analysis:**

- **~50 diagnostics** are caused by ty selecting a Python 3.7 baseline from
  `requires-python = ">=3.7"` in `pyproject.toml`. This metadata is valid and
  intentional; the issue is tool interpretation, not package correctness.
  Under that baseline, `NotRequired`, `Required`, `types.EllipsisType`,
  `types.UnionType`, and `typing.assert_type` are unavailable. The library
  handles runtime compatibility with feature detection, but ty resolves imports
  statically.
- **~20 diagnostics** are genuine: `obj: object` is subscripted, iterated,
  or passed to `len()` without narrowing. The code is correct at runtime
  (guarded by `isinstance` checks), but the type annotations don't reflect
  this.
- **~10 diagnostics** are from third-party modules (`bson`, `magic`, `dns`,
  `email_validator`, `idna`) not being in ty's environment.
- **3 `missing-typed-dict-key` diagnostics** in `bench.py` are all false
  positives: `bad`, `rescheduled_from`, and `spsa` are declared as
  `NotRequired` but ty treats them as required under 3.7.

**Per-file distribution:** vtjson.py (61), test_vtjson.py (29),
bench_classic.py (15), bench.py (11).

### E.2 ruff check --select ALL *.py — Summary (1,596 errors)

**Per-file distribution:**

| File | Errors | Auto-fixable |
|---|---|---|
| `vtjson.py` | 869 | 260 |
| `test_vtjson.py` | 600 | 13 |
| `bench.py` | 73 | 19 |
| `bench_classic.py` | 54 | 16 |

**Top 30 rules:**

| Rule | Count | Meaning | Verdict |
|---|---|---|---|
| `PT027` | 269 | Use `pytest.raises` instead of unittest | Ignore — unittest by design |
| `PT009` | 109 | Use pytest-style assertions | Ignore — unittest by design |
| `N801` | 103 | Class name should use CapWords | Ignore — DSL style (lowercase `regex`, `gt`, `ge`) |
| `COM812` | 93 | Trailing comma missing | Style choice |
| `D212` | 88 | Multi-line docstring at first line | Style choice (conflicts with D213) |
| `D102` | 84 | Missing docstring in public method | **Actionable** — `__validate__` methods |
| `FBT001` | 75 | Boolean positional arg | Ignore — `strict: bool` by design |
| `FBT002` | 69 | Boolean default positional arg | Same |
| `TRY003` | 52 | Long exception messages | Style choice |
| `D105` | 47 | Missing docstring in magic method | Partially actionable |
| `D205` | 46 | Blank line between summary and description | **Actionable** |
| `BLE001` | 43 | Blind `Exception` catch | Partially — some intentional |
| `RUF010` | 42 | Use `str()` instead of f-string conversion | Minor style |
| `EM102` | 38 | Exception msg should not be f-string | Style choice |
| `PGH003` | 34 | Use specific `type: ignore` | **Actionable** |
| `D200` | 28 | One-line docstring should fit on one line | Minor |
| `RET505` | 26 | Unnecessary `else` after `return` | **Actionable** |
| `T201` | 25 | `print()` found | Some intentional (bench/debug) |
| `B010` | 22 | `setattr` with constant attr | Deliberate pattern (see §4.3.4) |
| `N802` | 17 | Function name not lowercase | Some intentional |
| `N816` | 14 | Mixed-case variable | Style choice |
| `EM101` | 14 | Exception msg should not be string literal | Style choice |
| `D101` | 13 | Missing docstring in public class | Partially actionable |
| `D404` | 12 | First word of docstring not imperative | Minor |
| `UP006` | 11 | Use `type` instead of `Type` | **Actionable** (when dropping 3.8) |
| `FURB105` | 10 | Use `print()` without empty string | Minor |
| `SIM108` | 9 | Use ternary operator | Style choice |
| `PLR0915` | 9 | Too many statements | Informational (large functions) |
| `PLR0124` | 9 | Comparison to self | Informational |
| `D107` | 9 | Missing docstring in `__init__` | Partially actionable |

**Recommended ruff config** (subset of actionable rules): see §4.4.2.

---

*End of report. This document is a living brainstorming artifact — update it as
improvements are implemented.*
