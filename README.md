# signa.txn.group

The public website for SignA, served by GitHub Pages at
<https://signa.txn.group>.

GitHub Pages builds this repository with Jekyll from `main` at `/`. There is
no build step to run before pushing.

## Layout

- `index.html` is the landing page.
- `_layouts/default.html` is the shared page shell, with the sidebar for guides.
- `_data/nav.yml` lists the guides in the sidebar.
- `assets/` holds the stylesheet and icons.
- The `*.md` files are the user and dApp guides.

## Where the guides come from

The guides are copies of `docs/` in the private `txngroup/signa` repository.
Two differ on purpose, because this site is public and the source is not:

- `zip-installation.md` opens with how to get a build rather than how to make
  one.
- `dApp-guide.md` drops the "Trying it" section, which needs the repository.

`extension-developers.md` (building from source) is not copied.

When a guide changes in `signa`, copy it here and keep those differences.

## Previewing locally

```sh
docker run --rm -v "$PWD":/site -w /site -p 4000:4000 ruby:3.3 bash -c \
  'gem install github-pages webrick && jekyll serve --host 0.0.0.0'
```

Then open <http://localhost:4000>.
