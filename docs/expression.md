# Expression Engines

JFTL expressions are evaluated by **expression engines**. Each engine provides its own syntax, capabilities, and security model.

Expressions are selected using the following syntax:

```text
$ENGINE=expression
```

For example:

```text
$py=price * quantity
$pyeval=sum(values)
$pyrun=return _.upper()
```

If no engine name is specified, the template's configured default expression engine is used.

```text
$=price * quantity
```

The default engine is configured in the template's `config` section.

```json
{
  "config": {
    "default_expr_engine": "py"
  }
}
```

---

## Available Engines

| Engine | Safety | Language | Typical Use |
|---------|---------|----------|-------------|
| `nav` | Safe | JFTL Navigation | Data lookup only |
| `strict` | Safe | JFTL Navigation (strict) | Data lookup that must fail loudly on type mismatches |
| `py` | Safe | Restricted Python (SimpleEval) | Calculations, filtering, comprehensions |
| `pyeval` | Unsafe | Full Python `eval()` | Trusted templates requiring unrestricted expressions |
| `pyrun` | Unsafe | Full Python statements | Complex logic that requires loops, assignments, or multiple statements |

---


## nav

The `nav` engine is the simplest expression engine. It performs only navigation through the current document and runtime variables. See `navigation.md` for full syntax.

Note: The Navigation engine is automatically assumed when the expression starts with '$', and follow the syntax of navigation expressions (specifically, $, $^, $foo, $["bar"], ...). Usually, there is no need to specify it explicitly.

No functions, operators, or calculations are supported.

Typical uses include:

- Reading values
- Selecting nested fields
- Looking up variables
- Dynamic object indexing

### Examples

Current object:

```text
"$nav=$"
```

Nested field:

```text
"$nav=$.customer.name"
```

Variable lookup:

```text
"$nav=$total"
```

Dynamic lookup:

```text
"$.fxrate[$currency]"
```

*(The `.name` dot segments above are JFTL's own navigation grammar — see `navigation.md` — not Python attribute access, so they stay as-is even though the Python-engine examples below use bracket notation.)*

### Characteristics

- Safe
- No function calls
- No arithmetic
- No string manipulation
- Fastest expression engine
- A missing key, an out-of-range index, or navigating into a value of the wrong container type (e.g. a `.field` segment on an array) all resolve quietly to `Missing`, never an error. Use `strict` when that silence is unacceptable.

---

## strict

The `strict` engine uses the exact same navigation syntax as `nav`, with one difference: a **type mismatch** along the path — for example applying a `.field` segment to something that isn't an object, or an `[index]` segment to something that isn't an array — produces a runtime error instead of quietly resolving to `Missing`.

A simple missing key or a value that genuinely doesn't exist still resolves to `Missing` under `strict`, exactly as it does under `nav`. What `strict` catches is navigating as if the shape of the data were something it isn't.

### Example

Input:

```json
{ "tags": ["red", "green", "blue"] }
```

```text
"$nav=$.tags.owner"      →  Missing     (tags is an array, not an object — nav stays silent)
"$strict=$.tags.owner"   →  runtime error (tags is an array, not an object — strict reports it)
```

### Characteristics

- Safe
- Same grammar and capabilities as `nav` — no functions, arithmetic, or string manipulation
- Fails loudly on container-type mismatches instead of returning `Missing`
- Useful wherever a wrong-shaped document should be treated as a bug, not just an absent value

---

## py

The `py` engine evaluates expressions using a restricted subset of Python implemented with **SimpleEval**.

It is the recommended engine for most templates.

JSON objects are plain Python `dict`s, and `py`'s attribute-access whitelist only covers a handful of `str` methods — so descend into JSON objects/arrays with `['key']` / `[index]`, not `.key`. Single quotes are used for these Python string literals throughout this document, since the enclosing template value is itself a JSON string — using single quotes inside means the Python code never needs to escape a `"`.

Supported features include:

- Arithmetic
- Comparisons
- Boolean operators
- List comprehensions
- Dictionary comprehensions
- Basic built-in functions

Access to arbitrary Python objects and methods is restricted.

### Example 1

```text
"$py=price * quantity"
```

### Example 2

```text
"$py=customer['age'] >= 18"
```

### Example 3

```text
"$py=[item['name'] for item in _['items'] if item['price'] > 100]"
```

### Built-in Functions

The default configuration provides common functions including:

- `abs`
- `all`
- `any`
- `bool`
- `chr`
- `float`
- `int`
- `len`
- `max`
- `min`
- `ord`
- `range`
- `round`
- `sorted`
- `str`
- `sum`

Strings expose a limited set of methods, including:

- `lower`
- `upper`
- `strip`
- `lstrip`
- `rstrip`
- `startswith`
- `endswith`
- `replace`
- `split`
- `join`

### Characteristics

- Safe
- Good performance
- Recommended for most templates
- Supports comprehensions
- Does not allow arbitrary Python execution

---

## pyeval

The `pyeval` engine evaluates the expression using Python's built-in `eval()`.

The expression has unrestricted access to Python language features and any objects available in the evaluation environment. JSON objects are still plain `dict`s, so use `['key']` to descend into them — dot access only works for genuine Python attributes/methods (e.g. `.values()` on a dict, `.upper()` on a string), never for JSON fields.

**This engine should only be enabled for trusted templates.**

### Example 1

```text
"$pyeval=sum(_.values())"
```

### Example 2

```text
"$pyeval=[c for _, c in sorted((c['balance'], c) for c in customers)]"
```

Sorts `customers` by `balance` using a decorate-sort-undecorate pattern — `pyeval` does not permit `lambda` expressions (see `pyrun` below), so a `key=lambda c: ...` argument isn't available here.

### Example 3

```text
"$pyeval={name: value for name, value in _.items() if value > 0}"
```

### Characteristics

- Unsafe to use on untrusted templates
- Full Python expression syntax
- Arbitrary function calls permitted
- `lambda` expressions are **not** permitted (rejected at compile time)
- Suitable only for trusted environments

---

## pyrun

The `pyrun` engine executes Python statements rather than a single expression.

Unlike the other engines, it allows:

- assignments
- loops
- conditional statements
- early returns

The result of the expression is the value returned by the `return` statement.

The engine expect a valid python program. This means that python rules must be followed: each statement should be on its own line (use "\n" to enter new lines into the JSON), and indentation rules must be followed. As an alternative, Python allows multiple statements on a single line, separated by semicolon.

A compound statement (`if`, `for`, `while`) whose entire body is a single simple statement can similarly be kept on one line, by writing the statement directly after the header's `:` instead of indenting it on its own line:

```text
$pyrun=total = 0
for item in _['items']: total += item['price']
return total
```

Python's walrus operator (`:=`) is also available, letting a script bind an ad-hoc variable in the middle of an expression instead of needing a separate assignment statement first:

```text
$pyrun=values = [n for x in _['items'] if (n := x['price']) > 0]
return sum(values)
```

Here `n := x['price']` both computes and names each item's price once, reusing it for the filter without recomputing it.

As with `pyeval`, JSON objects are plain `dict`s — use `['key']` to descend into them, not `.key`.

**This engine should only be enabled for trusted templates.**

### Return Value

A `pyrun` script must reach an explicit `return` to produce a result. If execution runs to the end of the script without hitting a `return` — for example, an `if` with no covering `else` — the expression evaluates to `Missing`, the same as any other JFTL value that couldn't be produced. Normal `fallback` handling applies, so pair a `pyrun` expression with `fallback` whenever a script might legitimately fall through without returning.

### Example 1

```json
{
    "main": {
        "$": true,
        "data": { "user": { "first": "Jon", "last": "Doe" }},
        "out": "$pyrun= greet='Hello'; return greet + ' ' + _['user']['first']"
    }
}
```

will print the "Hello Jon"

### Example 2

```python
"$pyrun=\n
total = 0\n
for item in _['items']:\n
    total += item['price']\n
return total\n
"
```

### Example 3

```python
$pyrun=
result = []

for customer in customers:
    if customer['balance'] > 0:
        result.append(customer['name'])

return result
```

###


### Characteristics

- Unsafe
- Full Python statements
- Supports loops and assignments
- Lambda expressions supported
- Most flexible engine
- Slower than expression engines due to execution overhead

---

# Choosing an Engine

| Requirement | Recommended Engine |
|-------------|--------------------|
| Read data | `nav` |
| Read data, fail loudly on wrong-shaped input | `strict` |
| Calculations | `py` |
| Filtering and comprehensions | `py` |
| Advanced Python expressions | `pyeval` |
| Multi-statement algorithms | `pyrun` |

For most templates, **`py`** provides the best balance between functionality, performance, and safety. The `pyeval` and `pyrun` engines are intended only for trusted environments where unrestricted Python execution is acceptable.
