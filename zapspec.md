# Zappr Specification

**Version:** 0.2.0
**Ownership:** Zappr is owned by AXIOM
**Status:** In development.
**License:** Apache-2.0
**Last updated:** 2026-11-27

---

## 1. Overview

Zappr is a compiled, fully native programming language. The tool is `zap`: one binary that is compiler, build driver, runner, package manager and formatter at once. There is no separate compiler entry point and no tooling split — `zap` is the only way in.

Zappr yoinks the good parts of C, C++, TypeScript, Rust, Go and the rest, and leaves the bad parts. It compiles to Linux and Windows, all native. Native UI is built into the language, not bolted on. Configuration is one `Zappr.yaml`. Compile goal: **under 5 seconds for a full build**, incremental builds faster.

```
zap build          → one executable per platform
```

The compiler is written in Rust (edition 2024), permanently. Zappr is not self-hosted and is not intended to be. The compiler is an axiom tool; axiom tools are Rust.

---

## 2. Naming

| Name | Role |
|------|------|
| `Zappr` | The language |
| `zap` | CLI: compiler, driver, runner, package manager, formatter — one binary |
| `.znz` | Source file extension ("Zappr, not Zig") |
| `Zappr.yaml` | The one config file |
| `zapYaml` | The read-only namespace exposing `val:` entries in code |
| `use` | The namespace keyword for reaching other files and dependencies |

---

## 3. Goals and Principles

Zappr follows the axiom philosophies:

1. **No sidecars.** Companion files and helper processes count as bugs. One artifact per platform, nothing transient is ever written into a user project, no runtime files, no `.so` shims. If `zap` cannot produce one clean artifact, it fails with an error. It does not produce partial output.
2. **Native, not a wrapper.** Zappr drives its own codegen, its own linker configuration, its own everything. Zappr's build pipeline never delegates compilation, linking or verification to external tools — no gcc, no rustc, no chained or wrapped tooling, no verify-via. **One tool: zap.** This law governs zap's pipeline, not programs: a program built *by* zap may call whatever std functions it wants (§20) — that is program behavior, not tool chaining.
3. **Minimal configuration.** One `Zappr.yaml`, easy to read, edit and parse. Unknown keys are errors; defaults exist but are never silent (see §19).
4. **Signed by default.** Every build artifact is Ed25519-signed; `zap verify` checks it.
5. **One artifact per platform.** Linux gets one executable. Windows gets one executable. How static those executables are is principle 6's business, not this one's.
6. **Static linking.** Only base system libraries (libc, libm, libgcc and the loader) remain dynamic.
7. **Unix-y.** Familiar commands, fast, composable.
8. **Detect or error — never guess.** License matching, config keys, artifact shapes: if zap cannot know, it refuses.
9. **Blazing fast.** Under 5 seconds for a full build. The compiler is parallel, incremental and cache-first by design, not by tuning.

---

## 4. Architecture

```
zap (one Rust binary)
├── lexer          → tokens from .znz
├── parser         → AST
├── checker        → types, exclusivity, bounds, naming law, tidiness law, no-shadowing
├── call-collector → transitive closure of calls (the whole inclusion story, §12)
├── codegen        → artifact assembly: reachability packing + host stamping + signing (§18.1, §12)
├── linker config  → static binaries, Ed25519 signing
└── ui runtime     → native renderer
```

- **Parallel by module.** Independent `.znz` files compile concurrently, Go-style.
- **Incremental and content-hashed.** A touched file recompiles alone; caches live in the system temp dir, never in the project.
- **One visibility domain.** std, dependencies and app code get identical treatment. There is no "trusted std" tier — see §12.
- **The checker is fast by construction, not by tuning.** Structural types need no cross-module unification: a module checks against the flattened signatures of what it calls, in parallel, and a touched file re-checks alone. Exclusivity checks (§15.1) are per-function local. There is no whole-program type fixpoint anywhere in the pipeline.
- **No backend to build.** zap compiles to itself: an artifact is the zap host binary stamped with the packed, reachable source of your program (§18.1) and run in-process at startup. Nothing else — no LLVM, no Cranelift, no external toolchain — stands between your source and the machine.

---

## 5. Lexical

### 5.1 Files

- Source files are `.znz` ("Zappr, not Zig").
- **Filenames under `zap/` are lowercase — never camelCase.** `userService.znz` is an error; `userservice.znz` is correct.
- **Comments:**
  - `///` is a single-line comment, ends at the newline.
  - `//` opens a block comment; `\\` on its own line closes it (the "double line" shape). The first `\\` closes. No nesting. An unterminated `//` block is an error. `//` is never a one-liner.
- **Semicolons are optional.** A newline ends a statement; `;` compiles fine if written.

### 5.2 Naming law

Compiler-enforced, not linted:

| Form | Names | Example |
|------|-------|---------|
| `camelCase` | variables, functions, fields, use-names | `maxRetries`, `fetchUser` |
| `PascalCase` | types, consts | `User`, `MaxRetries` |
| `lowercase` | files and folders under `zap/` | `userservice.znz`, `http/` |

The casing *is* the mutability marker — see §6.2.

### 5.3 Tidiness law (errors, not warnings)

Zappr refuses to compile a junk drawer:

- Scattered, unorganized files at one folder level: **error**.
- Banned folder names — vague role-name drawers: `server/`, `utils/`, `misc/`, `helpers/`, `common/`, `lib/`. Name parts of your program (`http/`, `auth/`, `auth/session.znz`), not jobs.
- Encouraged shape: deep, specific, self-describing paths — `zap/thisPart/subPart/part.znz`.

---

## 6. Variables and Assignment

### 6.1 Declaration and reassignment

Assignment is always marked. `!` declares; `/` reassigns; bare `=` is a hard error.

```go
!x = 5            /// declare, inferred
!x: int = 5       /// declare, typed
/x = 6            /// reassign — x must already exist
/x+               /// increment (courtesy; /x = x + 1 is preferred)
/x-               /// decrement, same status
x = 6             /// COMPILE ERROR — unmarked assignment
```

The language has no implicit state changes: every mutation is visually loud, and a typo'd assignment (`lenght = 5`) is a compile error, never an implicit global.

### 6.2 Consts — the casing is the modifier

```go
!retries = 3        /// camelCase → variable, /retries = 4 is fine
!MaxRetries = 3     /// PascalCase → const; /MaxRetries = 4 is a COMPILE ERROR
```

There is no `const` keyword, no `let`/`var`, no `mut`. PascalCase names never rebind. Immutable-by-default falls out of the naming law.

### 6.3 No shadowing

A name is declared exactly once, ever, in its scope tree. Redeclaration — including in a nested scope — is a compile error. Loop variables and function parameters declare their names; nothing else may redeclare them.

---

## 7. Types

### 7.1 Growable numerics

| Type | Default | Opt-in fixed form |
|------|---------|-------------------|
| `int` | arbitrary precision, grows by value on the fly — never overflows, never wraps | `i8` `i16` `i32` `i64` `u8` `u16` `u32` `u64` |
| `float` | grows precision on the fly | `f32` `f64` |

Fixed-width types are chosen by hand when perf or wire formats demand it; the compiler holds you to the choice. Mixing coerces toward the bigger side (`int + i32` just works); real narrowing is an explicit cast (`as u8`) that truncates because you said so.

### 7.2 Strings

`str` is the default: owning, growable, refcounted. `"hi" + name` just works — there is no `.to_string()` battle. `&str` is the opt-in borrowed slice for zero-copy work, declared by hand.

### 7.3 Tuples

Tuples are unlimited in arity. They are the language's composition primitive: parameter lists, returns, type-argument lists and optionals are all tuples.

### 7.4 Types are values

There is exactly one kind of thing in Zappr: values. Types are values — passable, returnable, storable. `null` is a **type**, not a value.

```go
!T = User               /// a type in a variable
!scores: List(int) = [] /// List takes a type, returns a type — generics are calls (§10)
```

### 7.5 Optionals and null

Optionals are `(T, null)` tuples — the pad form. `null` exists only in type positions and pad slots; there is no runtime null value to assign or dereference. The billion-dollar mistake never enters the language.

- The pad form `(User, null)` is always available and always explicit.
- `autoNull: true` in `Zappr.yaml` fills the gap for un-annotated values: optionality is inferred, padding is automatic, access auto-unwraps, and an absent value surfaces as `wrongReturnType` at first use (§8.1). autoNull never changes what the explicit pad form does.

### 7.6 Declaring your own types

A struct is a **named tuple** — the composition primitive of §7.3 with a name on it. Declared with the same assignment law as everything else, PascalCase by the naming law:

```go
!User = (name: str, age: int)        /// a type value, declarable in any scope
!ada = User(name: "ada", age: 36)    /// construct by field name
ada.name                             /// access by name — .0 and .1 work too, it is still a tuple
```

A **union** is a tuple of type values — `Circle | Rect` is sugar for the type tuple `(Circle, Rect)`. Construction picks a side; **`match`** reads it back, and match must be exhaustive:

```go
!Circle = (r: float)
!Rect   = (w: float, h: float)
!Shape  = Circle | Rect              /// sugar for the type tuple (Circle, Rect)

!s = Circle(r: 2.5)

match s {
  Circle as c { print.nl("circle, r {c.r}") }
  Rect as r   { print.nl("rect, {r.w}x{r.h}") }
}
```

- **Exhaustive or else.** Every variant must be covered — a missing one is a compile error, not a runtime surprise. A final `else` arm covers whatever you did not name, which is how you opt out of exhaustiveness.
- Binding is optional: `Circle { ... }` runs the arm without naming the payload.
- Variants may themselves be unions; match narrows left to right, so nesting composes.
- **fnType is not the union tool.** fnType (§8.1) checks a value against *one expected type* at a boundary — a mismatch is a `wrongReturnType` poison, never "the other variant". Unions narrow with `match`; error boundaries narrow with `.fnType`. The two never swap jobs.

**Collection literals:** `[]` is an empty list, `{}` an empty map (§10), `{k: v, k2: v2}` a populated map. Maps are `Map(K, V)` values; keys and values are plain expressions, evaluated where the literal sits.

### 7.7 Constant folding

Pure computation with all-constant arguments folds to its result when the module is parsed — before checking, before packing, before the host ever runs it (§20 row 1):

- Literal arithmetic (`2 + 3`, checked — overflow leaves the expression for the growable runtime path, §7.1), string concatenation of literals, unary negation and not, and casts of literals.
- Pure builtins over constant args: `len` of a string literal, `math.abs` / `math.min` / `math.max` / `math.sqrt` / `math.floor` / `math.ceil`.
- Side-effecting calls never fold, division by zero never folds, and everything else is left to run time. Folding happens once per module and rides the content-hashed AST cache (§4), so it costs nothing on untouched files.

---

## 8. Errors

The error channel is a returned value of a different type. No tuples-of-err, no exceptions, no `?`.

- `Err(...)` is the universal error value. A function errors by returning it instead of its usual shape.
- A function may return *whatever it wants* — there are no preset return types and no return annotations.

### 8.1 `.fnType`

`.fnType(T)` takes a type value and checks at **runtime**:

- **Match** — the value narrows to `T`, checked once, stays narrowed.
- **Mismatch** — the value becomes a `wrongReturnType` poison carrying the expected type, the actual value, and the origin location. **No panic, no crash.**

Using a poison value as data fails loudly at the use site, with the full story. This is the NaN model: mismatches become inspectable data; failures happen where the value is consumed. fnType is a boundary tool, not a union tool — a mismatch is never "the other variant"; unions narrow with `match` (§7.6).

```go
!user = fetchUser(3).fnType(User)     /// got Err back? user is now wrongReturnType — no crash
print.nl(user.name)                   /// LOUD failure here, with expected/actual/origin
```

### 8.2 Boundary handling

```go
!user = fetchUser(3)                  /// trust: prototyping, "it cannot fail here"
!user = fetchUser(3).fnType(User)     /// assert: checked use
if fetchUser(3).fnType(User) as u {   /// branch: handle it properly
  print.nl("hello {u.name}")
} else as e {
  print.errNl("failed: {e}")
}
```

Unchecked use is allowed; misusing an `Err` as data fails loudly at first use, never silently.

---

## 9. Functions

### 9.1 Shape

```go
add(a, b) {
  return a + b
}
```

- No return annotations, no preset types, no `-> View` badges. A function returns whatever it returns.
- Parameters bind positionally from the argument tuple.

### 9.2 Soft arity, both directions

The compiler delivers the argument tuple whole; declared parameters are the head. Surplus and shortage are both the callee's business:

| Case | Behavior |
|------|----------|
| Exact count | binds |
| **Fewer** | legal — unbound params become `missingArg` poison (fails loudly at first use, with param/call-site origin); `args.count` inspects |
| **More** | legal — `args.rest` holds the surplus; `args.count` inspects |

Policy is the function's: error out (`if args.count < 2 { return Err(...) }`) or deliberately ignore. Silent misinterpretation is the only banned behavior.

### 9.3 Poison values

`wrongReturnType` and `missingArg` share one design: a mismatch never detonates at the boundary; it becomes a value that fails at first use with origin info, and is assertable in tests like any value.

### 9.4 Built-ins

Flat, verb-first:

| Call | Does |
|------|------|
| `print.nl(...)` | print args, space-separated, newline |
| `print.raw(...)` | no newline, no added spaces |
| `print.err(...)` / `print.errNl(...)` | stderr variants |
| `print.fmt("hi {name}", name)` | `{name}` interpolation, compile-time checked, every type prints itself |
| `read(int age)` | declare-and-read, whitespace-delimited; bad input fails loudly |
| `readInto(x)` | read into an existing variable |
| `readDialogue(str name)` | prints `>: ` prompt, reads the full line |
| `args.list()` | raw argv as `List(str)` — program name excluded |
| `args.opts(defaults?)` | parsed options as a map, overlaid on `defaults` (§9.5) |
| `args.positional()` | non-option arguments, in order |

**Interpolation is a string-literal feature, not a print feature.** `{expr}` in *any* string literal interpolates from the enclosing scope, compile-time checked — `label.text = "count is {count}"` (§16.1) is the same machinery as `print.fmt`, which is just the flat verb for printing one.

### 9.5 Native arg parsing

Arg parsing is built in — no clap, no builder ceremony, no derive macros. `args` is an ambient namespace of three verbs; the result is ordinary values you already know how to handle.

```go
main() {
  !o    = args.opts({name: "world", verbose: false})
  !rest = args.positional()        /// ["deploy", "prod"] — order preserved
  !raw  = args.list()              /// ["deploy", "--name", "world", "prod"]

  print.nl("deploying {o.name} {rest}")
  if o.verbose == true { print.nl("loud") }
}
```

`zap app deploy prod --name world --verbose --out=a.txt`

The grammar is deliberately tiny — everything clap does that you actually use, nothing it does that you don't:

| Form | Meaning |
|------|---------|
| `--name value` | option with a value |
| `--name=value` | same, one token |
| `-n value` | short option, single dash |
| `--flag` | boolean option, value `true` |
| `-f` | short boolean flag |
| `--` | everything after is positional, verbatim |
| `-` alone, `--x` inside a value | positional, untouched |

Rules:

- **No schema, no registration.** There is no `Arg::new("name")` to forget. `args.opts()` parses what it sees; unknown options land in the map like known ones. Whether an option *should* exist is app policy.
- **Defaults are just a map.** `args.opts({name: "world"})` is clap's `default_value` without the ceremony — parsed options overlay the defaults map, so `o.name` is always there. Without defaults, an absent option reads as the null side of the pad (§7.5) and surfaces through the same §8.1 machinery as every other optional.
- **Values are strings, flags are bools.** `--out a.txt` gives `o.out == "a.txt"`; `--verbose` gives `o.verbose == true`. Repeating an option keeps the last value — silent overwrite is data, not misinterpretation.
- **`args.positional()` and `args.opts()` partition `args.list()`.** Every token lands somewhere exactly once; nothing is dropped, reordered, or guessed.
- **Custom zap CLI commands get the same `args`.** `zap stamp --build 3` parses `--build 3` inside `stamp()` exactly as a built binary would — one parser, one grammar, everywhere args exist.
- Parsing is native and allocation-light: one pass over argv, no regex, no reflection, no proc macros — it fits the §21 budget trivially.

---

## 10. Generics

Types are values (§7.4), so generics need no special grammar. A generic is a function that takes types and returns a type; instantiation is indistinguishable from a call.

```go
!scores: List(int) = []
!cache: Map(str, User) = {}
!b = Box(int, str, User)

first(list: List(T)) {     /// T is PascalCase — type parameter names follow the naming law
  return list.0
}
```

- **No `<T>`.** Angle brackets are comparison operators; the paren form is the only parameterization mechanism.
- **Structural bounds.** The body needs `.0` on `T`, so the call site's type must have `.0`. No traits, no impl blocks, no coherence rules. Errors land at the call site, where they can be fixed.
- **Compile once per instantiation actually called.** The call-based inclusion rule (§12) already guarantees no unused instantiations exist.
- **Single-arg instantiations pad:** `List(int, null)`. There is no `..=`-style convenience anywhere — inclusive bounds are written by hand as `$(maxInclusive + 1)`.

---

## 11. Control Flow

### 11.1 Loops — five forms, one keyword

```go
for (x, y) in z { ... }            /// tuple-loop: per-element destructure; (k, v) for maps
for cond { ... }                   /// while
for { ... }                        /// infinite; break / continue as usual
for i in $(10) { ... }             /// count-loop: exactly n times, i starts at 0
for (!i = 0; i < n; /i+) { ... }   /// C-style; the header obeys the !/ / law — no exemptions
```

- **There is no range operator.** No `..`, no `0..10`. Counting is `for i in $(expr)` where expr is any number expression: a literal, a value, `list.len()`, `userCount + 5`.
- Inclusive counting is written by hand: `$(maxInclusive + 1)` — the `..=` equivalent, visible where it happens.
- Offset ranges compose: `for i in $(end - start) { !x = i + start }`.
- Growable `int` means absurd bounds are slow, never wrong.

### 11.2 Conditionals

```go
if count > 10 {
  print.nl("big")
} else {
  print.nl("small")
}
```

`if`/`else` are the language's own — UI and errors reuse them; there is no template dialect.

---

## 12. Modules and Inclusion

### 12.1 Files and namespaces

- Std is **ambient**: `print`, `read`, `nav`, `env` just exist. Never imported.
- `use.depName` opens a namespace — written **mid-file**, wherever first needed. No header imports, nothing to scroll past.
- Folders namespace **by fusion**: the file's path segments (lowercase folders + lowercase filename) fuse into one camelCase name. One dot, ever — no dot-ception:

```
zap/widgets.znz           →  use.widgets
zap/gui/widgets.znz       →  use.guiWidgets
zap/http/auth/session.znz →  use.httpAuthSession
```

- The filesystem stores structure; code sees a single flat identifier. Renaming or moving a file is a mechanical use-path rewrite (see §18.3).
- **No access modifiers.** A file's top-level names are reachable by anyone who can name its use-path — there is no `pub`/`private`/`internal`. Encapsulation is structural: helpers a caller shouldn't touch live nested inside a function or one folder deeper, where the tidiness law (§5.3) wanted them anyway.

### 12.2 Inclusion is calls, nothing else

- The unit of inclusion is the **call** — transitive: what a call calls, and what those call.
- References are free. `!x = json` loads nothing until `x.something(...)` is invoked.
- Position is irrelevant — a call on line 500 pulls in its dep exactly as hard as one on line 5.
- What you never call does not exist in the binary, at any depth. Dead-code elimination is not a pass; it is the default state.

### 12.3 One visibility domain

std, dependencies and app code are treated identically: same call-based inclusion, same build-time folding, same runtime specialization. There is no std-is-magic tier; std is dependency #1.

---

## 13. Dependencies

No registry. No lock file. GitHub is the registry, `Zappr.yaml` is the lock — and it locks **content**, not names.

```go
# Zappr.yaml
githubDeps:
  json:  github:someone/jsonlib          /// branch — floats
  http:  github:someone/httplib@1.4.2    /// tag
  flags: github:someone/flagkit@a1b2c3d  /// exact commit

resolved:                                /// written by zap on first fetch — do not hand-edit
  json:  someone/jsonlib@c4d1aa/t1b3d90  /// commit / SHA-256 of the tree zap got
  http:  someone/httplib@8c0f2b/t5e9011
  flags: someone/flagkit@a1b2c3d/t9f2e77
```

- **Pinned by content, not by name.** On first fetch, zap resolves the ref — branch, tag or sha — to an exact commit, hashes the source tree it received, and records both under `resolved:`. Every build after that replays the recorded commit and verifies the hash. Tags are pointers, not promises: a moved tag or rewritten branch never changes what a build replays. The ref is where you looked; the commit is what you got.
- **Verify, never hope.** Every build re-hashes the cloned (or cached) source and compares it to `resolved:`. Mismatch is a hard error — principle 8. Upstream rewrites and poisoned caches are detected, not absorbed.
- **The lock is flat.** Deps bring their own `githubDeps`; zap resolves the full transitive graph and records every package — direct and transitive — in the app's one `Zappr.yaml`. No nested locks, no sidecar lock file. Two deps demanding different commits of the same package is an error you resolve by hand, never a silent diamond. **The lock counts repos, not names:** two dep names resolving to the same repo are the same rule — they must agree on one commit, aliases included.
- **Unpinned branches float — legibly.** Each build records exactly what it fetched, so upstream drift is a diff of your `Zappr.yaml`, reviewable like code.
- Source is cloned into a shared cache outside the project (caches are zap's business, not sidecars), keyed by commit.
- **A dep is source code, nothing more.** It brings no scripts, no build steps, no toolchain. It cannot execute anything except being compiled.
- **Deps get no build-time privileges.** A dependency's `cli:` commands never register in your build, and its `val:` entries stay its own — build-time code execution is reserved for the app's own registered commands (§19.2). Dep code running inside your build would be the supply-chain hole; zap does not open it.
- Zappr runs one tool: zap. No chaining, wrapping or verifying-via external tools.

---

## 14. Parallelism

Structured concurrency, built in — Go's cheap tasks without Go's leaks, no async/await anywhere:

```go
main() {
  parallel {
    !images = fetchImages(zapYaml.imageDir)
    !metadata = loadMetadata()
  }                       /// join: all children done or one failed. No orphans, guaranteed.
  print.nl(images.len, metadata.title)   /// images/metadata in scope here — parallel publishes them

  !(tx, rx) = channel(int)
  spawn worker(tx)
  for value in rx {       /// channels drain until closed
    print.nl(value)
  }
}
```

- Tasks are cheap; spawn millions.
- `parallel { }` is a join point — nothing spawned inside outlives it.
- **Failure:** the first child to fail fails the block. Siblings are cancelled at their next safe point — the join still guarantees no orphans. There is no exception to throw, so the failure becomes data: every published name holds the same `Err(...)` value, carrying the failing child's origin. Poison rules as usual (§9.3): unchecked use fails loudly at first use; checked use is the §8.2 branch, naming the type you expect:

```go
parallel {
  !images = fetchImages(zapYaml.imageDir)
  !metadata = loadMetadata()
}
if images.fnType(List(str)) as imgs {
  print.nl(imgs.len, metadata.title)     /// success — all published names are real
} else as e {
  print.errNl("parallel block failed: {e}")
}
```
- **Scoping:** `parallel { }` is the one construct whose body declares *upward*. Its `!` declarations are published into the enclosing scope at the join — the names after the block are the names from inside it. No-shadowing counts them as enclosing-scope declarations: a clashing name already in the enclosing scope makes the block a compile error, and the block's names may not be redeclared after it.
- Data crossing a task boundary must be `Send`-safe: checked simply — values with shared mutable access do not cross (§15.1). Move semantics + ARC make this mostly free.
- **No function colors.** Any function can block, any function can spawn; parallelism never infects signatures. The runtime scheduler preempts/yields at safe points.

---

## 15. Memory

Move semantics + ARC. No garbage collector, and no Rust-style borrow checker — memory safety comes from ARC keeping referents alive, and borrow safety from the exclusivity law (§15.1).

- A borrow checker is a global dataflow analysis plus lifetime annotations. Zappr skips the analysis because ARC removes what it guards against: a referent cannot die while an alias exists, so dangling is impossible by construction. What remains — aliased mutation and data races — is the exclusivity law's job, and it is local.
- Values move by default; shared ownership is refcounted; cycles are the programmer's explicit problem (`weak` references exist for when you mean it). `weak(v)` wraps a shared value; reading it yields the `(T, null)` pad — an expired reference reads as the null side, never a dangling pointer.
- UI state and elements are ordinary values with ordinary ARC lifetimes — the state-lifetime tiers are §19.1.

### 15.1 The exclusivity law

Rust earns borrow safety with a global dataflow analysis over the whole function graph. Zappr enforces the same rule locally:

**Overlapping access to the same storage admits at most one writer — or any number of readers — never both.**

- **Moves, compile-time and local.** A moved value's binding is dead; using it is a compile error at the use site. The check never looks past the current function.
- **Exclusivity, compile-time where the borrows are visible.** If the checker sees two live borrows of the same storage and one writes — a loop body mutating the collection it iterates, aliased arguments to a mutating call — that is a compile error, reported at the borrow sites. Per-function, parallel with everything else, no fixpoint.
- **A runtime guard where borrows escape.** Borrows that leave the checker's sight — into a closure, a stored field, another task — carry a per-object access flag the compiler inserts. A violating overlap fails loudly as `exclusiveAccess` at the moment of the second access, with both origins; it never corrupts memory. Static checking is the common case; the escape hatch costs one flag set and clear.
- **Task boundaries stay Send-safe (§14).** Values with shared mutable access do not cross threads, so the runtime guard never has to arbitrate a race it cannot see.

Net effect: Rust's memory-safety guarantees — no dangling, no use-after-free, no data races, no iterator invalidation, no aliased mutation — with zero lifetime annotations and zero global analysis. That is the sense in which it is borrow-checker-esque: the same law Rust enforces, enforced where it is cheap instead of everywhere. The "fast by construction" claim in §4 leans on exactly this — per-function local checks scale with the code you touched, not the code you have.

---

## 16. Native UI

### 16.1 Imperative model

UI is plain code. Elements are values; you build them, wire events, and change properties. There is no declarative tree, no `!{ }` blocks, no view functions, no reactivity graph, no virtual DOM.

```go
main() {
  !win = ui.window("demo", 800, 600)
  !label = win.add.text("count is 0")
  !btn = win.add.button("increment")
  !count = 0

  onClick(btn) {
    /count+
    /label.text = "count is {count}"   /// you change the value, the screen follows
  }

  ui.run(win)
}
```

One rule: **mutate an element's property and it updates.** Setting `.text` patches exactly that node in the native renderer. No `getElementById`, no innerHTML, no re-render.

### 16.2 Rendering law

- **Always repainting.** The loop renders on every event-loop pass at the display's refresh cadence (a non-blocking pump), so state changes appear at max frame rate. There is no dirty gating and no sleeping.
- **`frameRate: auto`** (default) — the app mirrors the device's *actual* frame production, exactly, absorbing its jitter, tracking changes mid-run. A number (`frameRate: 144`) is a hard lock.
- **`efficientRedraw`** (default `false`) — when on, the renderer diffs each frame against what is actually on screen and redraws only the changed regions: a moved element repaints its old spot and its new spot, nothing else. No gating, no dirty tracking — every frame still renders, but only damaged areas are uploaded and composited. Off by default; the full-frame path is the reference behavior.

### 16.3 Routes

Built in, no dependency:

```go
main() {
  route("/", Home)
  route("/settings", Settings)
  route("/users/{id}", UserPage)       /// path params bind like arguments
}
```

- Routes are extensionless. A URL ending in `.html` does **not** match a route declared without it. A route explicitly named with `.html` (`route("/docs/v1.html", ...)`) is legal.
- Navigation is `nav.go("/settings")`.

---

## 17. Project Layout

```
my-app/
├── Zappr.yaml          /// the one config
└── zap/                /// all source lives here
    ├── main.znz        /// the entry point — exactly one, at zap/ root
    ├── gui/
    │   └── widgets.znz
    ├── http/
    │   └── routes.znz
    └── tests/
        ├── test.znz    /// the orchestrator — zap test runs this
        └── fetchuser/
            ├── happypath.znz
            └── rejectsnegative.znz
```

- **`main.znz`**: exactly one, at `zap/` root — duplicates are an **error**. It orchestrates: calls the pieces in order, wires startup, gets out. A logic-heavy `main.znz` earns a nudge; its end state reads like the app's README.
- **Tests** live in `zap/tests/`. `test.znz` orchestrates the full pipeline of group folders (`fetchuser/`, `gui/`). The assertion toolkit is `assert`, `.fnType` and the poison values — error cases are just values you check.
- **Tidiness is law** (§5.3): scattered files and banned folders are compile errors.

---

## 18. The zap CLI

```
zap new my-app            /// scaffold project + Zappr.yaml
zap run                   /// build (in-memory where possible) + execute
zap build                 /// linux / windows — via flags or targets in Zappr.yaml
                           /// --release for dist/release, default dist/debug
zap check                 /// type-check only, fast feedback
zap test                  /// run zap/tests/test.znz and its pipeline
zap fmt                   /// the janitor (§18.3)
zap verify                /// check an artifact's Ed25519 signature
zap copyright headers     /// stamp license headers, on explicit intent only
```

### 18.1 Build outputs

Artifacts land in `dist/`, named after the project, split by profile:

```
dist/debug/<name>          /// zap build        (-O0 -g)
dist/release/<name>        /// zap build --release  (-O2)
dist/debug/<name>.exe      /// --target=windows
```

- **linux / windows:** one static executable per platform. License headers ride inside the binary as a comment section. The native renderer is statically linked Rust: Wayland (wl_shm + xdg-shell, real server-side decorations) on Linux; X11 is not used. Windows uses Win32 GDI. The executable is the zap host binary (the full Rust toolchain runtime) with the project source tree packed into a `ZAPP` footer archive; at startup the host extracts it to a content-keyed cache dir and runs `main` in-process. No external compiler, linker or runtime is involved at build or run time.

### 18.2 Signing

`zap build` signs every artifact Ed25519 by default (`sign: true` in `Zappr.yaml`, `signKey` locates the key). `zap verify` checks a signature before you run something you downloaded.

### 18.3 zap fmt — the janitor

`zap fmt` is a formatter, organizer and splitter with explicit consent (`y/n/a`: yes to this one, no, or all remaining):

```
zap fmt
  add copyright headers to 14 files? (y/n/a)
  reorganize 3 scattered files? (y/n/a)
  split services.znz (1,204 lines) into 4 files? (y/n/a)
```

- **Headers:** stamps the copyright header (fuzzy-matched LICENSE, §18.4) into files missing them. Idempotent.
- **Organize:** moves scattered files into proper folders and dissolves banned drawers — proposed names are shown in the prompt and require approval or edits before anything moves; **fmt never invents names on its own** (§3.8: detect or error, never guess). When files move, every affected `use.` path is mechanically rewritten, since the path→name mapping is a pure function.
- **Split:** giant files (soft threshold ~500 lines) are offered for topic-based splitting — never mechanical line-counting. `main.znz` that grew fat gets dissolved back into topic files.

### 18.4 Copyright headers

- `zap copyright headers` runs on **explicit intent only** — nothing touches files during build.
- zap fuzzy-matches the project's `LICENSE` file against known licenses (same mechanism as GitHub: hash, similarity score, confidence threshold). No match → hard error; `license:` in `Zappr.yaml` overrides detection.
- Headers are idempotent and stale headers are updated in place.
- Headers survive compilation: as a comment section inside native binaries. The license travels with the artifact.

---

## 19. Zappr.yaml

One file. Every key optional except where noted; unknown keys are errors; defaults are never silent.

```go
# Zappr.yaml
name: my-app                /// required
version: 0.2.0              /// required

license: MIT                /// optional — overrides LICENSE fuzzy-matching
licenseHolder: axiomwtf     /// optional — for zap copyright headers

targets:                    /// optional — default is the host
  - linux
  - windows

stateTtl: 10m               /// optional — WARNING if omitted (§19.1)
frameRate: auto             /// optional — auto or a number (§16.2)
autoNull: false             /// optional — ergonomic optional handling (§7.5)
efficientRedraw: false      /// optional — damage-rect redraws only (§16.2)

sign: true                  /// optional — Ed25519 signing, default true
signKey: ./signing.key      /// optional — signing key location

val:                        /// build-time values; read verbatim as zapYaml.x
  appName: Zappr Demo
  maxRetries: 3

githubDeps:                 /// §13
  json: github:someone/jsonlib

resolved:                   /// written by zap on first fetch (§13) — verified every build, never hand-edited

cli:                        /// custom zap commands, written in Zappr (§19.2)
  - tools/stamp.znz
```

### 19.1 UI state lifetime

Component state has a three-tier lifecycle, automatic, no annotations:

| Tier | Condition | Where it lives |
|------|-----------|----------------|
| Hot | on screen / in use | the active render tree |
| Cold | navigated away, idle | plain RAM — survives, so returning restores your place |
| Gone | unused past `stateTtl` | removed; revisiting starts fresh |

`stateTtl` defaults to **10 minutes**, LRU eviction order, and **zap warns when it is not declared** — silent behavior is a sidecar. Values that must never die do not belong in component state; they belong in app-level state or `val:`.

### 19.2 val and the custom zap CLI

- Everything under `val:` is app data, read verbatim as `zapYaml.any_field` (yaml casing untouched: `zapYaml.any_field_here`). Nested structures are allowed; `zapYaml` is a read-only tree.
- **vals are immutable from within Zappr code.** The only mutation path is a **custom zap CLI** — project-registered commands (`cli:` in `Zappr.yaml`) written in Zappr, running in-process with build privileges:

```go
/// tools/stamp.znz — runs as `zap stamp`, with build privileges
stamp() {
  val.buildNumber = env.get("CI_NUMBER")
  val.builtAt = clock.nowIso()
}
```

Anything after the command name is that command's argv — `zap stamp --build 3` parses through the same `args.*` verbs as a built binary (§9.5).

- Code consts (PascalCase in `.znz`) are truly immutable; `val:` entries are immutable at runtime and mutable only by a custom zap CLI at build time.

| | Code consts (`!MaxRetries = 3`) | yaml vals (`val:`) |
|---|--------------------------------|--------------------|
| Immutable at runtime | yes | yes |
| Immutable during build | yes | no — custom zap CLI only |
| Naming law | PascalCase enforced | verbatim |
| Access | `MaxRetries` | `zapYaml.appName` |

---

## 20. Call Flattening

Because the compiler sees every call with full semantics, in one visibility domain, known calls **flatten**: the compiled program performs what the call was going to do, not the call.

| Case | Behavior |
|------|----------|
| Pure computation, known args (`math`, string ops, json of literals) | folds to its result during compilation (§7 constant folding) |
| Known semantics + known args (`thisShell.shellOut`, file IO) | dispatches at run time through the host's direct builtin — the operation itself, wrapper layers removed |
| Dynamic args | same direct builtin path; only the values are unknown until run time |

```go
!sha = thisShell.shellOut("git rev-parse HEAD")
```

runs as a direct `git` exec plus stdout capture: the host's `shellOut` builtin spawns the program directly. It does **not** go through a `/bin/sh` wrapper that parses the command string first (that fallback exists only for commands with shell metacharacters). That is the whole trick: where a naive toolchain emits call-upon-wrapper-upon-parse, zap performs the work, once, directly.

The same flattening removes repetition anywhere the compiler can see it: repeated known-shape calls collapse to their direct forms, wrapper layers around wrapper layers disappear, and external dependencies flatten identically — they ship as source, so their semantics are as visible as std's.

---

## 21. Performance Budget

- **Under 5 seconds** for a full build; incremental much faster.
- Parallel compilation by module; incremental content-hashed caches in the system temp dir.
- Generics monomorphize per-called-instantiation only (§10); the inclusion rule guarantees no unused instantiations.
- No backend of its own: the artifact is the host binary plus packed reachable source (§18.1); nothing to tune, nothing to link.
- Binary size = sum of what you called. Nothing else.
- Precompiled module state; no repeated import-graph re-parsing.

---

## 22. Non-Goals

- No self-hosting. The compiler is Rust, permanently.
- No package registry, no lock file, no dependency network beyond GitHub.
- No borrow checker — meaning no Rust-style lifetime annotations and no global borrow analysis; the exclusivity law (§15.1) provides the same guarantees locally. No garbage collector, no async/await, no function coloring.
- No template language for UI, no JSX, no virtual DOM.
- No LLVM backend in v1.
- No sidecar files of any kind — if a feature requires one, the feature is wrong.
