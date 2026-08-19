# Overview

> Draft placeholder — content to be written (docs #1 on the 0.4.0 launch
> checklist).

## What is JFTL?

JFTL (JSON Fold Template Language) is a declarative JSON-to-JSON
transformation engine. A template is itself JSON; applying a template to an
input document produces an output document.

Design philosophy:

- **Explicit over implicit** — reserved names, sigils, param kinds, and
  predicate categories all use closed, documentable vocabularies.
- **Loud failures over silent ones** — compile errors are never deferred to
  runtime.
- **Portability as a hard constraint** — every language feature is
  evaluated against implementability across five target languages (Python,
  Java, Node/JS, Perl, and lower-priority .NET/Go).
- **Verbosity as a feature** — no feature is added without a concrete,
  recurring use case.

## Template structure

*TODO: `defs`, `main`, `datasets`, `config` — top-level shape.*

## The component system

*TODO: component call syntax `{"$": "name", ...args}`, `function` vs
`macro`, two-phase compilation.*

## Navigation and expressions

*TODO: `$.foo` navigation, `${...}` interpolation, `$py=`/`$pyeval=`/
`$pyrun=` expression engines.*

## Logic statements

*TODO: `set`, `check`, `case`, `foreach`, `transform`, `fallback`/`error`.*
