# CLI

> Draft placeholder — content to be written (docs #2 on the 0.4.0 launch
> checklist).

`jf-template` applies a JFTL template to one or more input files.

```
jf-template template [file1 file2 file3 ...]
```

- `template` — path to the template JSON file.
- `file1, file2, ...` — input JSON files, processed independently (a
  failure on one does not stop the others, with `-k`/`--keep-going`).
- No input files given — read a single input from stdin.
- No arguments at all — read the *template* from stdin, render with no
  input data.

## Options

*TODO: document each flag (`-f/--input-format`, `-D/--data`,
`-F/--dataset`, `--split`, `-t/--target`, `-s/--sections`, `-m/--map`,
`-k/--keep-going`, `-q/--quiet`, `-v/--verbose`, `--indent`, `--raw`,
`-N/--no-plugins`, `-A/--all-plugins`, `--enable`, `-e/--entry`) with
examples.*

## Exit codes

*TODO: table of `ExitCode` values and what each means.*
