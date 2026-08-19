# jftl-lang

Specification and documentation for JFTL (JSON Fold Template Language),
versioned independently of any particular engine implementation.

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

The `CNAME` file itself is written to the root of the `gh-pages` branch by
the deploy workflow (see `.github/workflows/docs.yml`) — it is intentionally
**not** placed under `docs/`, since `mike` builds each spec version into
its own subfolder and a `docs/CNAME` would end up nested per-version
instead of at the root, where GitHub Pages actually requires it.

## Versioned deploys

Docs are built and deployed per spec version using
[`mike`](https://github.com/jimporter/mike), via the manually-triggered
"Build and deploy docs" GitHub Actions workflow. The `latest` alias is
updated deliberately, not automatically on every push.

To deploy locally (requires push access to `gh-pages`):

```bash
mike deploy --push --update-aliases 0.4 latest
```

## Structure

```
docs/
  index.md          # landing page
  overview.md        # docs #1 — concepts, template structure
  cli.md              # docs #2 — jf-template usage
  cookbook.md         # docs #3 — worked examples, recipes
  spec/                # normative, versioned language specification
    namespacing.md      # reserved namespaces (jftl./std.), resolution order
```
