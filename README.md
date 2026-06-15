<p align="center"><img src="https://raw.githubusercontent.com/go-compressions/brand/main/social/go-compressions.png" alt="go-compressions/docs" width="720"></p>

# go-compressions/docs

Versioned documentation for [go-compressions](https://github.com/go-compressions),
built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and
versioned with [mike](https://github.com/jimporter/mike). Published to the
`gh-pages` branch and served at <https://go-compressions.github.io/docs/>.

The organization landing page
([go-compressions.github.io](https://go-compressions.github.io)) links here.

## Local preview

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
mkdocs serve                       # http://localhost:8000 (current sources)
mike serve                         # preview the versioned site
```

## Releasing a new docs version

```bash
mike deploy --push --update-aliases <version> latest
mike set-default --push latest
```
