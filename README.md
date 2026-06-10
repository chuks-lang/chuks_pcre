# @chuks/pcre

A **PCRE-style backtracking regular-expression engine, written 100% in Chuks.**

Chuks's built-in `std/regex` is RE2 — linear-time and safe, but (exactly like Go's
standard library) it has **no lookaround and no backreferences**. `@chuks/pcre` is
the backtracking counterpart: the advanced engine shipped as a package, the same way
[`dlclark/regexp2`](https://github.com/dlclark/regexp2) sits outside Go's std. No
native shim, no C, no WASM — pure Chuks that runs identically on the VM and the AOT
native backend.

It's built to power things RE2 can't: running **TextMate grammars** for syntax
highlighting, lint rules with backreferences, validators with lookahead, and so on.

## Install

```sh
chuks add @chuks/pcre
```

## Quick start

```chuks
import { Regex, Match } from "pkg/@chuks/pcre"

var re = new Regex("(\\w+)@(\\w+)\\.(\\w+)")
var m = re.find("contact ada@chuks.dev now", 0)
if (m != null) {
    println(m.matched())   // ada@chuks.dev
    println(m.group(1))    // ada
    println(m.group(2))    // chuks
    var sp = m.span(1)     // [8, 11]  (rune offsets)
}
```

## Features

| Category        | Supported                                                              |
|-----------------|------------------------------------------------------------------------|
| Literals & dot  | `abc`, `.` (any char except newline), escaped metachars `\.` `\*` …    |
| Escapes         | `\d \D \w \W \s \S` · `\n \t \r \f \v \0`                              |
| Char classes    | `[a-z]`, `[^...]`, ranges, shorthands inside `[\d.]`, literal `-`       |
| Quantifiers     | `* + ?`, `{n} {n,} {n,m}`, and lazy `*? +? ?? {n,m}?`                   |
| Groups          | capturing `(…)`, non-capturing `(?:…)`, named `(?<name>…)`              |
| Alternation     | `a|b|c`                                                                 |
| Anchors         | `^ $`, `\b \B`, `\A \z`, `\G` (search-start anchor — TextMate's friend) |
| Backreferences  | numeric `\1`…`\99`, named `\k<name>`                                    |
| Lookaround      | lookahead `(?=…)` `(?!…)`, lookbehind `(?<=…)` `(?<!…)`                 |
| Groups (extra)  | atomic `(?>…)` (≈ non-capturing), possessive `a++ a*+`                  |
| Inline flags    | `(?i)` `(?i:…)` case-insensitive · `(?x)` `(?x:…)` extended/verbose (whitespace + `#` comments ignored) · `m s U` parsed |
| Hex / unicode   | `\xHH`, `\x{H…}`, `\uHHHH`                                              |
| POSIX classes   | `[[:alnum:]]` `[[:digit:]]` `[[:space:]]` … inside `[ … ]`              |

## API

### `class Regex`

```chuks
new Regex(pattern: string)              // compiles the pattern (throws on a syntax error)

re.find(str: string, startPos: int): Match?   // leftmost match at/after startPos, or null
re.test(str: string): bool                    // does it match anywhere?
re.findAll(str: string): []Match              // all non-overlapping matches, left to right
re.source(): string                           // the original pattern
re.groupCount(): int                          // number of capturing groups
```

`find` takes a **start position** (a rune index), so you can scan a string left to
right without re-slicing it — which is exactly what a tokenizer needs. `\G` anchors
to that start position.

### `class Match`

```chuks
m.matched(): string            // the whole match (group 0)
m.start(): int                 // start rune offset
m.end(): int                   // end rune offset
m.group(i: int): string        // text of capture group i (0 = whole match), "" if absent
m.span(i: int): []int          // [start, end] rune offsets of group i, [-1,-1] if absent
m.groupByName(name): string    // text of a named group
m.groupCount(): int            // number of capturing groups
```

## How it works

A recursive-descent **parser** turns the pattern into an AST, and a
**continuation-passing backtracking matcher** walks it: each node tries every way it
can match from a position and calls a continuation for the rest of the pattern,
unwinding on failure. Capture spans live in a shared array that's saved and restored
across backtracking, so a repeated group reports its last iteration just like PCRE.

Like any backtracking engine, pathological patterns (`(a+)+$` against a long
non-match) can backtrack exponentially. If you don't need lookaround or
backreferences, prefer the linear-time built-in `std/regex`.

## Tests

```sh
chuks run tests/index.test.chuks      # VM
chuks build tests/index.test.chuks -o /tmp/t && /tmp/t   # AOT
```

70 assertions, green on both the bytecode VM and the AOT native backend.

## License

MIT
