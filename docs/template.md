# Template Structure

Every JFTL template is a JSON document. At the top level, the template may contain up to four entries:

| Entry | Required | Description |
|--------|----------|-------------|
| `main` | **Yes** | The main template to execute. |
| `defs` | No | Defines reusable components (functions/macros) callable from `main`. |
| `datasets` | No | Defines datasets bundled with the template. |
| `config` | No | Template-wide configuration options. |

Minimal template:

```json
{
  "main": {
    "message": "Hello World"
  }
}
```

---

# main

The `main` entry is the root of the template.

It may contain:

- literal JSON
- navigation expressions
- logic elements
- expression engine invocations
- any other valid JFTL construct

Example:

```json
{
  "main": {
    "name": "$.name",
    "age": "$.age"
  }
}
```

When rendered with

```json
{
  "name": "Alice",
  "age": 42
}
```

the result is

```json
{
  "name": "Alice",
  "age": 42
}
```

The engine always starts execution from `main`.

---

# defs

Templates may define reusable **components** — functions or macros — in the top-level `defs` entry. A component is compiled once and can be invoked from `main` (or from other components) with a call expression of the form `{"$": "componentName", ...params}`.

`defs` is a JSON object mapping each component name to its definition:

| Field | Required | Description |
|--------|----------|-------------|
| `type` | No | `"function"` (default) or `"macro"`. |
| `params` | No | Object mapping parameter names to a default-value expression. Use the literal value `"$_required"` to mark a parameter as **required** — it has no default, and must be supplied at every call site. |
| `out` | **Yes** | The compiled body of the component, evaluated each time the component is called. |

At a call site, only parameters listed in `params` are accepted; unknown parameters and missing required parameters are both reported as compile errors.

## Example 1 — no parameters ("include")

A component with no `params` is effectively a named, reusable snippet — call it wherever you'd otherwise repeat the same literal structure:

```json
{
  "defs": {
    "footer": {
      "out": {
        "company": "Acme Corp",
        "support_email": "support@acme.example"
      }
    }
  },

  "main": {
    "title": "Invoice",
    "footer": { "$": "footer" }
  }
}
```

Result:

```json
{
  "title": "Invoice",
  "footer": {
    "company": "Acme Corp",
    "support_email": "support@acme.example"
  }
}
```

## Example 2 — using `$_` (the caller's current data)

A component doesn't need formal parameters to see data — `$_` refers to whatever is the "current" value at the call site, the same value `$` would navigate from. This lets a component act on the caller's context implicitly:

```json
{
  "defs": {
    "summary": {
      "out": {
        "customer": "$_.name",
        "item_count": "$py=len(_.items)"
      }
    }
  },

  "main": { "$": "summary" }
}
```

Input:

```json
{
  "name": "Alice",
  "items": [ "pen", "notebook", "eraser" ]
}
```

Result:

```json
{
  "customer": "Alice",
  "item_count": 3
}
```

## Example 3 — two parameters, one required and one optional

```json
{
  "defs": {
    "price_tag": {
      "params": {
        "amount": "$_required",
        "currency": "USD"
      },
      "out": "$py=currency + ' ' + str(round(amount, 2))"
    }
  },

  "main": {
    "regular": { "$": "price_tag", "amount": 19.9 },
    "euro":    { "$": "price_tag", "amount": 19.9, "currency": "EUR" }
  }
}
```

Result:

```json
{
  "regular": "USD 19.9",
  "euro": "EUR 19.9"
}
```

`amount` has no default, so every call must supply it; omitting it is a compile error. `currency` has a default of `"USD"`, so it may be overridden per call, as in `"euro"` above.

### function vs. macro

`type` defaults to `"function"` when omitted. A `function` runs in its own frame, anchored back at the top of the template — it does not see the caller's local variables, only the parameters it was explicitly given (plus `$_`/the caller's current data). A `macro` instead runs inside a child of the *caller's* frame, so it transparently sees any variables the caller has `set`, in addition to its own parameters.

---

# datasets

Templates may embed datasets that become available during rendering through the built-in `_datasets` variable.

The `datasets` entry must be a JSON object mapping dataset names to arbitrary JSON values.

```json
{
  "datasets": {
    "countries": {
      "US": "United States",
      "CA": "Canada"
    },

    "colors": [
      "red",
      "green",
      "blue"
    ]
  },

  "main": {
    "country": "$_datasets.countries.US"
  }
}
```

During rendering the engine combines datasets from three sources, in the following order:

1. datasets embedded in the template
2. datasets registered by the application
3. datasets supplied explicitly for the render operation

Later sources override earlier ones when the same dataset name is used.

When using the CLI options `--dataset` (also `-F`) and `--data` (also `-D`) can add data to the engine. See `CLI.md`.

---

# config

The optional `config` object controls template-wide behavior.

Current configuration options are shown below.

| Option | Type | Default | Description |
|---------|------|---------|-------------|
| `default_expr_engine` | string | `""` | Default expression engine used by `$=...` expressions. |
| `drop_null_attributes` | boolean | `false` | Omits object attributes whose final value is `null`. |
| `action_tag` | string | `"$"` | Name of the object key that marks an object as a logic block, component call, or forced literal, instead of plain data. |

Example:

```json
{
  "config": {
    "default_expr_engine": "py"
  },

  "main": {
    "sum": "$=a + b"
  }
}
```

---

# default_expr_engine

Normally an expression explicitly specifies its engine:

```text
$py=a + b
$pyeval=a + b
$pyrun=
return a + b
```

When `default_expr_engine` is configured, it specify the engine that will be used when no engined is provided: 

```json
{
  "config": {
    "default_expr_engine": "py"
  },

  "main": {
    "total": "$=price * quantity"
  }
}
```

is equivalent to

```json
{
  "main": {
    "total": "$py=price * quantity"
  }
}
```

If no default engine is configured, `$=...` expressions have no engine associated with them and compilation fails unless one is specified explicitly.

---

# drop_null_attributes

By default, object members whose value evaluates to `null` remain in the output.

Template:

```json
{
  "main": {
    "name": "$.name",
    "phone": "$.phone"
  }
}
```

Input:

```json
{
  "name": "Alice",
  "phone": null
}
```

Output:

```json
{
  "name": "Alice",
  "phone": null
}
```

Setting

```json
{
  "config": {
    "drop_null_attributes": true
  }
}
```

produces

```json
{
  "name": "Alice"
}
```

Only object attributes are removed. Array elements remain in place even when their value is `null`.

---

# action_tag

`action_tag` controls which object key JFTL treats as the **action marker** — the key that turns a JSON object into a logic block, a component call, or a forced literal, rather than an ordinary data object.

By default this marker is `"$"`, so:

```json
{ "$": true, "check": "$.enabled", "out": "$.value" }
```

is a logic block, and

```json
{ "$": "greet", "name": "$.customer" }
```

is a call to the `greet` component.

Some teams prefer a more self-describing marker than a bare `$`. Setting `action_tag` to `"$do"` renames it everywhere — logic blocks, component calls, and forced literals are all written with `$do` instead of `$`:

```json
{
  "config": {
    "action_tag": "$do"
  },

  "defs": {
    "greet": {
      "params": { "name": "$_required" },
      "out": "$py='Hello, ' + name + '!'"
    }
  },

  "main": {
    "$do": true,
    "data": "$.people",
    "foreach": {
      "var": "person",
      "out": { "$do": "greet", "name": "$person.name" }
    }
  }
}
```

Input:

```json
{
  "people": [ { "name": "Alice" }, { "name": "Bob" } ]
}
```

Result:

```json
[ "Hello, Alice!", "Hello, Bob!" ]
```

The top-level `foreach` logic block is marked with `{"$do": true, ...}`, and each iteration calls the `greet` component with `{"$do": "greet", ...}` — exactly the same grammar as with the default `$` marker, just spelled `$do`.

`action_tag` only changes this object-key marker. It has no effect on the `$`-prefixed **string** grammar (`"$.foo"`, `"$py=..."`, navigation, interpolation, `"$$"` escaping, etc.), which always starts with a literal `$` regardless of this setting.

---

# Complete Example

```json
{
  "config": {
    "default_expr_engine": "py",
    "drop_null_attributes": true
  },

  "datasets": {
    "currency": {
      "USD": "$",
      "EUR": "€"
    }
  },

  "defs": {
    "price_tag": {
      "params": {
        "amount": "$_required",
        "currency": "USD"
      },
      "out": "$py=currency + ' ' + str(round(amount, 2))"
    }
  },

  "main": {
    "name": "$.customer",
    "symbol": "$_datasets.currency.USD",
    "total": "$=price * quantity",
    "total_display": { "$": "price_tag", "amount": "$=price * quantity" }
  }
}
```

This template:

- executes the `main` entry,
- exposes the `currency` dataset through `_datasets`,
- evaluates `$=...` expressions using the `py` engine by default,
- calls the `price_tag` component defined in `defs`, and
- removes object attributes whose final value is `null`.
