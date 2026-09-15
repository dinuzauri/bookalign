# BookAlign

BookAlign structurally aligns an original-language EPUB with its official
translation, then rebuilds the result as a bilingual EPUB for parallel
reading.

1. Extracts body segments from the original and translated books.
2. Uses multilingual embeddings and dynamic programming for local or
   chapter-level alignment.
3. Preserves the original book structure and writes the translation back by
   paragraph or sentence.

The project currently focuses on fiction, especially Japanese originals with
Chinese translations, while also supporting basic workflows for English,
Spanish, and other language pairs.

## Related project

[bilingual-epub-toolkit](https://github.com/StarryGuli/bilingual-epub-toolkit)
is a similar project by a friend. It uses Gale-Church alignment without GPU
computation and provides an online service. It is worth studying as a related
approach.

## Two ways to run BookAlign

This repository provides two execution paths:

1. Recommended: `skills/bookalign-labse`
2. Direct: the `uv run bookalign ...` CLI pipeline

Prefer the skill workflow.

- The skill uses a review-first process: check the environment, extract both
  books, inspect chapter and body drift, align local slices, and build only
  after review.
- It makes the riskiest parts of the production workflow explicit, including
  `chapter_id` consistency checks, `slice_plan`, and review of unmatched
  segments.
- It is more reliable than a one-shot CLI run for dirty EPUBs, front/back
  matter mixed into the body, shifted chapter numbering, and notes or comments
  accidentally included in the text.

The CLI pipeline remains useful when:

- the input books have clean structure;
- the chapter mapping is already known to be approximately correct; or
- you want a quick first EPUB or alignment JSON.

## Recommended environment

Basic requirements:

- Python `3.12`
- `uv`
- a project virtual environment or `uv run`
- a local LaBSE model path, or a usable `HF_TOKEN`

Recommended alignment backends:

- Prefer local CUDA with a local LaBSE model.
- Without a GPU, extraction, inspection, and chapter review can still be done;
  CPU alignment works but is considerably slower.
- If no local model is available, use Hugging Face inference only with an
  explicitly configured `HF_TOKEN`.

VRAM guidance:

- `8 GB+` is generally more comfortable for stable local LaBSE and
  chapter-level alignment.
- `4-6 GB` may work for small chapters or sliced runs, but is not recommended
  for whole-book alignment by default.
- With CPU only, use the staged skill workflow rather than the direct
  whole-book CLI.

Check the environment with:

```bash
uv run python skills/bookalign-labse/scripts/check_environment.py --json
```

The report includes:

- `recommended_backend`
- `recommended_model_name`
- `recommended_device`
- `preferred_local_model`

## Typical use cases

- Read a literary original and its official translation without switching
  between readers.
- Preserve a published translation instead of relying on machine translation.
- Save alignment results as JSON for separate builder or layout debugging.
- Review chapter and body drift manually before building the final EPUB.

## Installation

The project uses Python 3.12 and `uv`.

```bash
uv sync --group dev --group align
```

If LaBSE is already available locally, pass its path explicitly rather than
depending on first-run online model resolution.

## Recommended usage: skill workflow

When using this repository in an agent environment, prefer:

- [skills/bookalign-labse/SKILL.md](skills/bookalign-labse/SKILL.md)

The recommended process is:

1. Confirm `<python-entry>`, `<skill-root>`, the model path, remote inference
   policy, and the artifacts directory.
2. Run `check_environment.py`.
3. Extract both books.
4. Inspect `list_book_chapters`, `get_chapter_preview`, and
   `sentence_segments`.
5. Run a chapter-consistency self-check.
6. Create a `slice_plan` for clean slices.
7. Align one slice at a time.
8. Use `review_unaligned_segments(...)` to inspect all unmatched segments.
9. Export review artifacts, then build the final EPUB.

See the complete production workflow:

- [skills/bookalign-labse/references/production-workflow.md](skills/bookalign-labse/references/production-workflow.md)

## Direct usage: CLI pipeline

For books with clean structure, run:

```bash
uv run bookalign \
  "books/source.epub" \
  "books/target.epub" \
  "out/output.epub" \
  --source-lang ja \
  --target-lang zh \
  --model-name /path/to/LaBSE
```

The defaults are:

- `builder-mode=source_layout`
- `writeback-mode=paragraph`
- `layout-direction=horizontal`
- `device=cuda`

For a denser sentence-level interleaved reading version:

```bash
uv run bookalign \
  "books/source.epub" \
  "books/target.epub" \
  "out/output-inline.epub" \
  --source-lang ja \
  --target-lang zh \
  --model-name /path/to/LaBSE \
  --writeback-mode inline
```

To save alignment JSON for later builder-only work:

```bash
uv run bookalign \
  "books/source.epub" \
  "books/target.epub" \
  "out/output.epub" \
  --source-lang ja \
  --target-lang zh \
  --model-name /path/to/LaBSE \
  --alignment-json-output "out/alignment.json"
```

Rebuild later from that JSON:

```bash
uv run bookalign \
  "books/source.epub" \
  "books/target.epub" \
  "out/output-rebuilt.epub" \
  --source-lang ja \
  --target-lang zh \
  --alignment-json-input "out/alignment.json"
```

The CLI remains a relatively direct one-shot pipeline:

```text
source EPUB + target EPUB
-> filtered_preserve extraction
-> heuristic chapter matching
-> Bertalign alignment
-> EPUB build
```

It is suitable for clean inputs, but should not replace the staged review
workflow.

## Examples

The following images show sentence-level and paragraph-level alignment:

- `docs/images/kinkaku-inline.png`
- `docs/images/kinkaku-paragraph.png`
- `docs/images/harry-potter-paragraph.png`

![Kinkaku-ji sentence-level alignment](docs/images/kinkaku-inline.png)
![Kinkaku-ji paragraph-level alignment](docs/images/kinkaku-paragraph.png)
![Harry Potter paragraph-level alignment](docs/images/harry-potter-paragraph.png)

## Current features

- Preserves the source spine order and most of its body structure.
- Supports `paragraph` and `inline` writeback modes.
- Saves `AlignmentResult` as JSON for reuse and debugging.
- Retains TOC entries, notes, and front/back matter instead of mixing them
  directly into body alignment.
- Stores unmatched target chapters in JSON and can write them to appendix
  pages.
- Rewrites footnote references and backlinks to avoid dead note-page links.
- Adds two leading spaces to Chinese translation paragraphs during build.
- Supports `slice_plan`, unmatched-segment review, and review-artifact export
  through the skill workflow.

## Known limitations

- The most stable combinations are still Japanese or English fiction to
  Chinese translation.
- EPUB quality has a major effect: dirty TOCs, unusual footnotes, and
  fragmented XHTML can all reduce quality.
- CLI whole-book chapter matching remains heuristic and should not be treated
  as ground truth.
- `inline` mode has stricter source EPUB requirements and is less robust than
  `paragraph` mode.
- Poetry, formulas, captions, and image-heavy layouts are not current
  optimization targets.
- LaBSE-style multilingual models have higher VRAM, startup, and environment
  costs than ordinary scripts.

## Repository structure

```text
bookalign/
├── align/      # Alignment abstractions and Bertalign adapter
├── epub/       # EPUB reading, extraction, CFI, and builder
├── models/     # Shared Segment / AlignmentResult models
├── cli.py      # Command-line entry point
└── pipeline.py # End-to-end one-shot pipeline

skills/
└── bookalign-labse/   # Recommended review-first skill

docs/           # README images and supplementary documentation
scripts/        # Environment and runtime helpers
tests/          # pytest
```

## Documentation

- [Technical details, current direction, and boundaries](TECHNICAL.md)
- [Recommended skill production workflow](skills/bookalign-labse/references/production-workflow.md)

## Acknowledgements

This project directly benefits from these open-source projects:

- [Flow](https://github.com/pacexy/flow): used to render the screenshots in
  the README and examples.
- [Vecalign](https://github.com/thompsonb/vecalign): the original source of
  much of the project's alignment-algorithm exploration.
- [Bertalign](https://github.com/bfsujason/bertalign): the current alignment
  backend used by BookAlign.
- [calibre](https://github.com/kovidgoyal/calibre): an important reference for
  EPUB CFI behavior.

## Development

Run the full test suite:

```bash
uv run pytest
```

Run the core skill tests:

```bash
uv run pytest skills/bookalign-labse/tests/test_service_api.py skills/bookalign-labse/tests/test_builder_refactor.py -q
```

## Future directions

- Continue consolidating the skill-first staged production workflow.
- Improve sentence splitting and chapter matching across more language pairs
  and EPUB styles.
- Add more reliable automatic correction for local alignment drift windows.
- Add finer layout controls and reader-compatibility behavior to the builder.
- Move from an offline EPUB tool toward an in-reader parallel-reading
  component.
