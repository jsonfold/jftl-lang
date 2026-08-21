# User Variables

This page describes how JFTL stores and resolves user-defined variables at
render time: the namespace concept, how a Logic Statement creates a new
namespace, the three ways to put a variable into it, and the two ways to
read it back.

---

## 1. Namespaces (a.k.a. Frames)

At render time, every point in the template tree is evaluated against a
**Frame** (`RuntimeContext` in `model.py` / `Frame` in `core.py`). A Frame
is a namespace: it holds variables, and a link (`parent`) to the Frame it
was created from. Frames form a chain all the way back to the root Frame
created for the top-level `render()` call.

A Frame's variables come from two sources:

- **Built-ins** — a fixed set of system-provided values (`_input`,
  `_top`, `_datasets`, `_level`, and the rest documented in
  `variables.md`) that are present automatically, without any action from
  the template author.
- **User variables** — whatever the template itself adds via `set` or
  `foreach` (see §3 below).

```
root Frame  (built-ins: _input, _top, _datasets, ...)
   └── child Frame  (+ whatever "set"/"foreach" added)
          └── child Frame  ...
```

**Resolution rule:** looking up a variable named `foo` starts at the
*current* Frame and walks up the parent chain, stopping at the first
Frame that defines `foo` — whether that's a user variable or a built-in.
The value found there — closest scope wins — is the result. If no Frame
in the chain defines it at all, the lookup **fails** (an unbound-variable
error) rather than quietly resolving to `Missing` — see `variables.md`
for the distinction between that and genuine `Missing` values produced by
navigation.

This same resolution order is what the `pyrun`/`pyeval`/`py` expression
engines build their evaluation namespace from, walking the Frame chain
from the root down and letting closer scopes overwrite farther ones —
with one caveat for built-ins specifically, covered in §5.

A Frame is *not* created for every statement — only specific constructs
create one. Everything else (plain literals, navigation, interpolation,
`check`/`data` inside a Logic Statement, etc.) evaluates against the Frame it
was given.

---

## 2. A Logic Statement (`"$": true`) creates a new namespace

Every time a Logic element (`{"$": true, ...}`) is evaluated, it creates a
**new child Frame** before Stage 1 (`set`/`check`) runs:

- The new Frame starts with only the standard built-ins (see
  `variables.md`) — no user variables of its own yet, and nothing carried
  over bodily from the enclosing Frame.
- Its `parent` is the Frame the Logic Statement was evaluated in. User
  variables declared in that enclosing Frame — or any Frame further up
  the chain — are still reachable, just via the resolution walk in §1,
  not by being copied in.
- Any variables it defines live **only** in this new Frame — they are
  local to that Logic Statement and disappear once it finishes; they do
  not leak into the enclosing scope.
- A *nested* Logic Statement (e.g. as the value of a `set` entry) creates
  yet another child Frame, whose parent is this one — so it can still
  read everything the outer Logic Statement defined, via the chain.

---

## 3. Adding variables to the namespace

Within one Logic Statement's Frame, variables can be added in three ways.
All three write into the **same** Frame — the one created for that Logic
Statement — not into separate per-iteration sub-Frames.

### (1) `set`

```json
"set": { "VAR_NAME": "EXPR" }
```

Each entry is evaluated once, in Stage 1, and assigned into the current
Frame's namespace.

### (2) `foreach` — `var` and `key`

```json
"foreach": {
  "in": "$.items",
  "var": "item",
  "key": "idx"
}
```

- `var` names the variable bound to **each item's value** on every
  iteration. If omitted, the item is not bound to a name at all — it
  simply becomes the current value (`$`/`_`) for that iteration.
- `key` names the variable bound to the **item's key or index**. If
  omitted, it still defaults to `_key` — the loop always exposes the
  current key/index under that name unless you override it.

Both are (re)assigned on every iteration, in the same Frame, so later
iterations overwrite earlier ones — they hold the *current* item/key, not
a per-iteration snapshot.

### (3) `foreach` → `update`

```json
"foreach": {
  "in": "$.items",
  "var": "item",
  "out": "EXPR",
  "update": { "VAR_NAME": "EXPR" }
}
```

`update` is nested *inside* the `foreach` object. Each entry is
(re-)evaluated **after** the per-item `out`, once per iteration, and
assigned into the same Frame's namespace — making it the standard way to
build a running accumulator across iterations (the accumulator can then
be read by later iterations, or by the outer `out`/`case` stage once the
loop finishes).

---

## 4. Variable naming — the "id" format

Variable names must look like an identifier:

```
^[A-Za-z]\w*$
```

— a letter, followed by any number of letters/digits/underscores. This
rule is enforced identically everywhere a template introduces a variable
name: `set` keys, `foreach`'s `var`/`key`, and `foreach.update` keys.

Two things worth knowing about this rule beyond simple validation:

- **It's what keeps user variables from ever colliding with built-ins.**
  Every JFTL built-in (`_input`, `_level`, `_data`, ...) starts with an
  underscore, and this rule requires a variable name to start with a
  *letter* — so a user-declared variable can never accidentally shadow a
  built-in. It's a deliberate safety property, not just a naming
  convention.
- **It doesn't guarantee the name is safe in Python**, only that it's a
  syntactically legal identifier. A name like `class` or `return` passes
  this check but will fail to parse inside `$py=`/`$pyeval=`/`$pyrun=`,
  since those are genuine Python reserved words. Avoid them even though
  the grammar itself won't stop you.

---

## 5. Reading variables back

### Via navigation

```
$foo         → lookup a variable named "foo" on the current Frame chain
$foo.bar[0]  → same lookup, then walks .bar / [0] on the result
```

Note the distinction from data access: `$.foo` reads a field of the
*current data value* (`_`), while `$foo` (no dot) reads the *variable*
named `foo` from the namespace chain. `$^` reads the original top-level
input; `$<` reads the parent's current data; `$%` reads the Frame itself.

### Via Python expressions (`$py=`, `$pyeval=`, `$pyrun=`)

Every **user-declared** variable visible on the Frame chain (root to
current, closest wins) is exposed as an ordinary Python variable in the
expression's evaluation namespace, alongside `_` (current value) and
`_input`:

```json
"$py=foo + 1"
```

This guarantee is specifically for user-declared variables (`set`,
`foreach`'s `var`/`key`, `update`). Built-in system variables are fully
reliable via navigation (`$foo`) at every nesting level, but there's a
known limitation from inside `$py=`/`$pyeval=`/`$pyrun=`: some built-ins
resolve to their render-wide/top-level value there rather than the
current frame's — `_level`, for example, may read `0` from inside a
`$py=` expression even several logic elements deep, while `$_level` via
navigation correctly reads the actual current depth. If a built-in's
value matters inside a Python expression, prefer reading it via
navigation into a `set` variable first, then use that variable in the
expression.

---

## 6. Examples

### 6.1 Set a variable to a constant

```json
{
  "$": true,
  "set": { "greeting": "hello" },
  "out": "$greeting"
}
```

Input: (any) → Output: `"hello"`

### 6.2 Set a variable to a composite value

```json
{
  "$": true,
  "set": {
    "profile": { "name": "$.name", "id": "$.id" }
  },
  "out": "$profile"
}
```

Input: `{"name": "Ada", "id": 7}`
Output: `{"name": "Ada", "id": 7}`

### 6.3 Set a variable via an arithmetic Python expression

```json
{
  "$": true,
  "set": {
    "x": "$.a",
    "y": "$.b",
    "sum": "$py=x + y"
  },
  "out": "$sum"
}
```

Input: `{"a": 3, "b": 4}` → Output: `7`

(`x` and `y` were themselves just set via `set`, and are visible inside
`$py=` as plain Python names because they live in the same Frame.)

### 6.4 Set a variable to the result of another Logic Statement

```json
{
  "$": true,
  "set": {
    "label": {
      "$": true,
      "check": "$.active",
      "out": "Active",
      "fallback": "Inactive"
    }
  },
  "out": "$label"
}
```

Input: `{"active": true}` → Output: `"Active"`
Input: `{"active": false}` → Output: `"Inactive"`

The inner `{"$": true, ...}` gets its own child Frame (a child of the
outer Logic Statement's Frame) while it evaluates; only its final result
— not any variables it might itself define — is assigned to `label` in
the outer namespace.

### 6.5 `foreach` `var`/`key` plus `update` accumulator

```json
{
  "$": true,
  "set": { "running": 0 },
  "foreach": {
    "in": "$.nums",
    "var": "n",
    "key": "idx",
    "out": "$py=str(idx) + ':' + str(n)",
    "update": { "running": "$py=running + n" }
  },
  "out": "$running"
}
```

Input: `{"nums": [1, 2, 3]}` → Output: `6`

`n` and `idx` are rebound each iteration; `running` is updated after each
item's `out` and still holds its final value once the loop ends — which
is what the outer `out` returns, overriding the loop's own collected
per-item output.