# Getting Started

This page gets you from nothing to a rendered template in a few minutes. For the full grammar, see the Reference section; for the concepts and mental model, see the Overview.

---

## 1. Install

```bash
pip install jftl
```

This gives you the `jf-template` command-line tool.

---

## 2. Write a template

Templates are plain JSON files. Save this as `hello.json`:

```json
{
  "main": {
    "greeting": "$py='Hello, ' + _['name'] + '!'",
    "id": "$.id"
  }
}
```

`main` is the root of every template — see `template.md` for the full top-level shape (`main`, `defs`, `datasets`, `config`).

---

## 3. Provide input and render

Save this as `input.json`:

```json
{ "name": "Ada", "id": 7 }
```

Run it:

```bash
jf-template hello.json input.json
```

Output:

```json
{
  "greeting": "Hello, Ada!",
  "id": 7
}
```

With no input file, `jf-template` reads a single input document from stdin instead:

```bash
echo '{"name": "Ada", "id": 7}' | jf-template hello.json -
```

---

## 4. A template that does more than substitute values

```json
{
  "main": {
    "$": true,
    "data": "$.orders",
    "foreach": {
      "var": "order",
      "if": "$py=order['total'] > 100",
      "out": { "id": "$order.id", "total": "$order.total" }
    }
  }
}
```

Given:

```json
{
  "orders": [
    { "id": 1, "total": 50 },
    { "id": 2, "total": 150 },
    { "id": 3, "total": 200 }
  ]
}
```

this filters and reshapes in one step:

```json
[
  { "id": 2, "total": 150 },
  { "id": 3, "total": 200 }
]
```

This is a Logic Statement (`"$": true`) — the core construct behind everything beyond plain substitution. See `logic.md` for the full pipeline (`set`/`check`/`data` → `foreach` → `out`/`case` → `transform` → `fallback`/`error`).

---

## Next steps

- **`navigation.md`** — the `$.foo`, `$foo`, `$<`, `$^` path expressions used everywhere above.
- **`logic.md`** — the full Logic Statement pipeline, including the `map`/`filter` shortcuts.
- **`expression-engines.md`** — `$py=` (used above), plus `$pyeval=`/`$pyrun=` for more advanced cases.
- **`variables.md`** / **`uservar.md`** — built-in variables (`_input`, `_key`, ...) and how to declare your own with `set`.
- **`defs.md`** — writing reusable `function`/`macro` components.
- **Cookbook** — worked, task-oriented examples once it's written.
