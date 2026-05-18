# LOCMAF Internet Draft

Sources for the IETF Internet Draft on **LOCMAF — Low Overhead CMAF
for Media over QUIC**.

Companion site: <https://locmaf.dev> (repo: `Eyevinn/locmaf`).
Reference implementation: <https://github.com/Eyevinn/moqlivemock>.

## Build

This repo uses the [martinthomson/i-d-template][tpl] tooling as a
git submodule under `lib/`.

```sh
# one-time, after fresh clone
git submodule update --init

# render the draft to txt + html
make

# clean
make clean
```

Outputs (gitignored): `draft-einarsson-moq-locmaf.txt`,
`draft-einarsson-moq-locmaf.html`.

The source is [kramdown-rfc][kdrfc] Markdown
(`draft-einarsson-moq-locmaf.md`). `make` invokes `kdrfc` to produce
RFCXML v3 and then `xml2rfc` to render the final formats.

### Dependencies

- **Ruby** with `kramdown-rfc` (`gem install kramdown-rfc`)
- **Python** with `xml2rfc` (`pip install xml2rfc`)
- **idnits** for submission lint (`brew install idnits` or download
  from <https://tools.ietf.org/tools/idnits/>)

The Makefile bootstraps these into a local `.gems/` and `.venv/`
when run for the first time.

## Once you have a GitHub remote

```sh
git remote add origin git@github.com:Eyevinn/locmaf-id.git
git push -u origin main
make -f lib/setup.mk setup    # enables GitHub Pages, archive, CI
```

After `setup`, every push renders `draft-einarsson-moq-locmaf.html`
to GitHub Pages and a PR check runs `idnits`.

## Submitting to the IETF

1. `make submit` produces `draft-einarsson-moq-locmaf-00.xml`
   (version bump on each cut).
2. Upload at <https://datatracker.ietf.org/submit/>.
3. The Datatracker re-runs `idnits` and emails confirmation.

[tpl]: https://github.com/martinthomson/i-d-template
[kdrfc]: https://github.com/cabo/kramdown-rfc
