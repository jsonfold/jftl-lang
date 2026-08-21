# Transformations

This page describes the `transform` attribute of a Logic Statement: where it sits in the evaluation pipeline, the built-in transformations, and their input/output shapes and error conditions.

---

## Where `transform` sits in the pipeline

A Logic Statement (`{"$": true, ...}`) evaluates in stages:

1. Setup — `set`, `check`, `data`
2. Iteration — `foreach` (optional)
3. Output — `out` / `case`
4. Transformation — `transform` (optional)
5. Recovery — `fallback`, `error`

`transform` is a **named structural transformation** applied to the value produced by the `out` or `case` stage (which may itself depend on the result of `foreach`).

A Logic Statement may specify **at most one** transformation. To perform multiple structural transformations, simply nest Logic Statements.

```json
{ "$": true, "out": "EXPR", "transform": "merge" }
```

Two things are worth noting:

* **Transforms are skipped for `Missing`.** If no value was produced (for example, because `check` failed or a previous stage returned `Missing`), the transformation is bypassed and processing continues with `fallback`.
* **Transformation errors are recoverable.** If a transformation reports an error (for example, because the input has the wrong shape), the resulting notice continues through the normal pipeline and may still be handled by `error`.

---

# Built-in transformations

The built-in transformations fall into four categories.



## Built-in transformers, at a glance

| Name              | Input                              | Output | Description |
|-------------------|-------------------------------------|--------|-------------|
| `flatten`         | Array of Arrays                    | Array  | concatenate multiple arrays |
| `merge`           | Array of objects                   | Object | concatenate multiple objects |
| `object_to_pairs` | Object                             | Array of Pairs | Split an object into an array of `[key, value]` pairs |
| `pairs_to_object` | Array of `[key, value]` pairs      | Object | Build an object from `[key, value]` pairs |
| `kvs_to_object`   | Array of `{key, value}` entries    | Object | Build an object from `{"key": ..., "value": ...}` entries |
| `values_to_map`   | Object whose values are `[key, value]` pairs | Object | Build a lookup map from those pairs |
| `drop_missing`    | Array or object                    | Same   | Remove entries with `Missing` |
| `join`            | Array of scalars                   | String | concatenate stringified elements |

| Category            | Transformations                                          |
| -------------------- | -------------------------------------------------------- |
| Structural           | `flatten`, `merge`, `drop_missing`                        |
| Object conversion    | `object_to_pairs`, `pairs_to_object`, `kvs_to_object`     |
| Lookup construction  | `values_to_map`                                           |
| String construction  | `join`                                                     |

---

# Transformation reference

## Structural transformations

### `flatten`

**Description:** Concatenates an array of arrays into a single flat array. `null` sub-arrays are treated as empty and skipped.

**Common use case:** collapsing a `foreach` that produced one sub-array per item into a single combined array.

**Input:** an array whose elements are either `null` or arrays.

**Output:** a flat array preserving the original order.

**Errors:**

* `FLATTEN_INPUT` — top-level value is not an array.
* `FLATTEN_ITEM` — an element is neither `null` nor an array.

**Examples**

```json
{ "$": true, "out": "$.groups", "transform": "flatten" }
```

Input:

```json
{"groups": [[1,2],[3],null,[4,5]]}
```

Output:

```json
[1,2,3,4,5]
```

```json
{
  "$": true,
  "foreach": {
    "in": "$.orders",
    "var": "o",
    "out": "$o.items"
  },
  "transform": "flatten"
}
```

---

### `merge`

**Description:** Merges an array of objects into a single object. Keys from later objects overwrite keys from earlier ones. `null` entries are ignored.

**Common use case:** combining partial objects (defaults, overrides, or one object per iteration).

**Input:** an array containing objects or `null`.

**Output:** a single merged object.

**Errors:**

* `MERGE_INPUT`
* `MERGE_ITEM`

**Examples**

```json
{ "$": true, "out": "$.records", "transform": "merge" }
```

Input:

```json
{"records":[{"a":1},{"b":2},{"a":9}]}
```

Output:

```json
{"a":9,"b":2}
```

---

### `drop_missing`

**Description:** Removes entries whose value is the `Missing` sentinel.

**Common use case:** removing optional values generated during `foreach`.

**Input:**

* array
* object
* `null`
* `Missing`

**Output:** same container type with `Missing` entries removed.

**Errors:**

* `DROP_MISSING_INPUT`

**Examples**

```json
{
  "$": true,
  "foreach": {
    "in": "$.items",
    "var": "item",
    "out": "$item.optional"
  },
  "transform": "drop_missing"
}
```

---

# Object conversion

### `object_to_pairs`

**Description:** Converts an object into an array of `[key, value]` pairs.

**Common use case:** preparing an object for processing by `foreach`, or reshaping a lookup table before reconstructing it with `pairs_to_object`.

**Input:** object

**Output:** array of `[key, value]` pairs, preserving iteration order.

**Errors:**

* `TO_PAIRS_INPUT`

**Examples**

```json
{
    "main": {
        "$": true,
        "data": {
            "$": true,
            "data": "$_datasets.country",
            "transform": "object_to_pairs"
        },
        "foreach": {
            "out": {
                "country": "$_[0]",
                "currency": "$_[1]"
            }
        }
    },

    "datasets": {
        "country": {
            "USA": "US Dollar",
            "GBR": "British Pound",
            "JPY": "Japanese Yen"
        }
    }
}
```

Output:

```json
[
  {
    "country": "USA",
    "currency": "US Dollar"
  },
  {
    "country": "GBR",
    "currency": "British Pound"
  },
  {
    "country": "JPY",
    "currency": "Japanese Yen"
  }
]
```

---

### `pairs_to_object`

**Description:** Builds an object from an array of `[key, value]` pairs.

**Common use case:** reconstructing an object after processing its entries — as `[key, value]` pairs — with `foreach`.

**Input:** an array whose elements are `[key, value]` pairs (2-element arrays, with the key as a string).

**Output:** object.

**Errors:**

* `TO_OBJECT_INPUT` — top-level value is not an array.
* `FROM-PAIRS-ITEM` — an element is not a well-formed 2-element `[key, value]` pair.

A pair whose key isn't a string is silently dropped from the result rather than reported as an error.

**Examples**

```json
{
    "$": true,
    "out": "$.pairs",
    "transform": "pairs_to_object"
}
```

Input:

```json
{
    "pairs": [
        ["a",1],
        ["b",2]
    ]
}
```

Output:

```json
{
    "a":1,
    "b":2
}
```

---

### `kvs_to_object`

**Description:** Builds an object from an array of explicit `{"key": ..., "value": ...}` entries — the object-shaped counterpart to `pairs_to_object`'s array-pair entries.

**Common use case:** reconstructing an object after processing its entries — as `{key, value}` objects — with `foreach`.

**Input:** an array whose elements are two-attribute objects, each with a string `"key"` and a `"value"`.

**Output:** object.

**Errors:**

* `FROM_KV_INPUT` — top-level value is not an array.
* `FROM_KV-ITEM` — an element isn't a well-formed `{"key": ..., "value": ...}` entry, or its `"key"` isn't a string.

**Examples**

```json
{
    "$": true,
    "out": "$.entries",
    "transform": "kvs_to_object"
}
```

Input:

```json
{
    "entries": [
        { "key": "a", "value": 1 },
        { "key": "b", "value": 2 }
    ]
}
```

Output:

```json
{
    "a": 1,
    "b": 2
}
```

---

# Lookup construction

### `values_to_map`

**Description:** Builds a lookup table by extracting key/value pairs from an existing collection.

Unlike `pairs_to_object`/`kvs_to_object`, which reconstruct an object directly from its own entries, `values_to_map` is intended for building **transition maps** that allow efficient conversion between identifiers.

**Common use cases**

* ISO code → currency
* Currency → country
* Employee ID → employee record
* Product code → product definition

**Input:** an object whose values are two-element `[key, value]` arrays. (The object's own top-level keys are not used in the output — only the `[key, value]` pairs found in its values are.)

**Output:** an object mapping every extracted key to its corresponding value.

If duplicate keys are generated, the last occurrence wins.

**Errors:**

* `VALUES_TO_MAP_INPUT`
* `VALUES_TO_MAP_ITEM`

### Example — Building transition maps

```json
{
    "main": {
        "$": {
            "set": {
                "db": {
                    "US": { "code": "USA", "country": "United States", "currency": "USD", "captial": "Washington DC" },
                    "UK": { "code": "GBR", "country": "Great Britian", "currency": "GBP", "captial": "London" },
                    "JP": { "code": "JPN", "country": "Japan", "currency": "JPY", "captial": "Tokyo"}
                },

                "currency_of": {
                    "$": true,
                    "data": "$db",
                    "foreach": {
                        "out": [ "$.code", "$.currency" ]
                    },
                    "transform": "values_to_map"
                },

                "country_of": {
                    "$": true,
                    "data": "$db",
                    "foreach": {
                        "out": [ "$.currency", "$.country" ]
                    },
                    "transform": "values_to_map"
                },

                "globus": {
                    "$": true,
                    "data": "$db",
                    "foreach": {
                        "out": [ "$.code", "$" ]
                    },
                    "transform": "values_to_map"
                }
            }
        },

        "JP currency": "$db.JP.currency",
        "currency of USA": "$currency_of.USA",
        "country of JPY": "$country_of.JPY",
        "Capital of UK": "$globus.GBR.captial"
    }
}
```

The generated transition maps can then be used through ordinary navigation expressions, providing efficient lookups throughout the remainder of the template.

---

# String construction

### `join`

**Description:** Joins an array of scalar values into a single string.

`null` becomes `"null"`, booleans become `"true"` or `"false"`, and numbers are converted to their string representation.

**Common use case:** building display strings or composite identifiers without requiring an expression engine.

**Input:** array of scalar values.

**Output:** string.

**Errors:**

* `JOIN-STR-TYPE`

**Examples**

```json
{
    "$": true,
    "out": "$.parts",
    "transform": "join"
}
```

Input:

```json
{
    "parts": [
        "Hello, ",
        "World",
        "!"
    ]
}
```

Output:

```json
"Hello, World!"
```

```json
{
    "$": true,
    "out": [
        "Count: ",
        "$.count",
        " (",
        "$.active",
        ")"
    ],
    "transform": "join"
}
```

Input:

```json
{
    "count": 3,
    "active": true
}
```

Output:

```json
"Count: 3 (true)"
```