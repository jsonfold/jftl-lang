# jftl-lang

Specification and documentation for JFTL (JSON Fold Template Language).

## License

No license is currently granted — all rights reserved. See
[`COPYRIGHT`](COPYRIGHT). This repository is public for reference purposes
only; public visibility does not itself grant any rights to reuse,
reproduce, or build upon its contents.

Implementations (Python reference, Java, Node/JS, Perl, ...) declare
compatibility against a spec version here, e.g. "compatible with
jftl-lang 0.4".

## Local development

```bash
pip install -r requirements.txt
mkdocs serve
```

## Custom domain (jftl-lang.dev)

GitHub Pages is configured to serve this site at `jftl-lang.dev`. Two
one-time setup steps (not automated, do these once):

1. **DNS** — at your domain registrar, add DNS records pointing at GitHub
   Pages:
   - `A` records for the apex domain (`jftl-lang.dev`) → GitHub's Pages IPs:
     `185.199.108.153`, `185.199.109.153`, `185.199.110.153`,
     `185.199.111.153`
   - (Optional) `CNAME` record for `www.jftl-lang.dev` → `YOUR_ORG.github.io`
     if you also want the `www` subdomain to resolve.
2. **GitHub repo settings** — Settings → Pages → Custom domain →
   enter `jftl-lang.dev` → Save. Enable "Enforce HTTPS" once GitHub
   finishes issuing the certificate (can take a few minutes after DNS
   propagates).

The `CNAME` file lives under `docs/` and is copied to the site root
automatically by `mkdocs build`/`mkdocs gh-deploy` — no extra step needed.

## Deploying

The site is a single tree (currently at spec version 0.4 — see
`docs/index.md`). Deploy via the manually-triggered "Build and deploy
docs" GitHub Actions workflow, or locally (requires push access to
`gh-pages`):

```bash
mkdocs gh-deploy --force
```

## Structure

```
docs/
  CNAME               # custom domain (jftl-lang.dev)
  index.md            # landing page
  overview.md          # docs #1 — concepts, template structure
  cli.md                # docs #2 — jf-template usage
  cookbook.md           # docs #3 — worked examples, recipes
  spec/                  # normative language specification
    namespacing.md        # reserved namespaces (jftl./std.), resolution order
```
