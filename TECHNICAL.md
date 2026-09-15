# BookAlign Technical Direction

This document describes BookAlign's current implementation, why the
repository keeps both a pipeline and a skill workflow, and which direction is
currently recommended.

## 1. Current recommendation

BookAlign has two practical execution paths:

1. The `bookalign` CLI one-shot pipeline.
2. The `skills/bookalign-labse` review-first workflow.

The second path, skill-first, is currently recommended.

This is not because the CLI is broken. Real EPUB noise is much more complex
than clean parallel text:

- front matter can shift all body chapters;
- `chapter_id` ownership and sentence-level records are not always stable;
- a visible chapter can contain notes, commentary, leftover TOC text,
  chronology, or appendices; and
- some books can only be aligned safely in slices rather than as a whole.

The technical direction has therefore changed from a single whole-book
pipeline:

```text
environment check
-> extraction
-> chapter / sentence consistency review
-> mixed-content inspection
-> explicit slice planning
-> local alignment per slice
-> unmatched-region review
-> final EPUB build
```

The CLI remains useful for clean books, quick experiments, initial alignment
JSON, and builder-only regression work.

## 2. Responsibilities of the two paths

### CLI pipeline

The direct orchestration path is centered on:

- `bookalign/cli.py`
- `bookalign/pipeline.py`

It remains a one-shot flow:

```text
source EPUB + target EPUB
-> filtered_preserve extraction
-> heuristic chapter matching
-> Bertalign alignment
-> EPUB build
```

Its chapter matching should be understood as a heuristic suggestion layer, not
as production-grade ground truth.

### Skill workflow

The recommended workflow is documented in:

- `skills/bookalign-labse/SKILL.md`
- `skills/bookalign-labse/references/workflow.md`
- `skills/bookalign-labse/references/production-workflow.md`

It explicitly requires:

- confirming the interpreter, model path, remote inference policy, and
  artifacts directory;
- checking the environment before choosing a backend;
- inspecting chapters before trusting `chapter_id`;
- running a consistency self-check before chapter alignment; and
- sending only clean slices to production building.

This is the workflow that best matches real-world book data.

## 3. Core data model

### `Segment`

Defined in:

- `bookalign/models/types.py`

`Segment` is the smallest working unit in the system. It is both an alignment
input and a positional anchor for writeback.

Important fields:

- `text`: sentence or paragraph text;
- `cfi`: EPUB CFI for the unit;
- `paragraph_cfi`: containing paragraph anchor;
- `chapter_idx / paragraph_idx / sentence_idx`: structural position;
- `raw_html`: original block-level HTML;
- `text_start / text_end`: sentence character range within its paragraph;
- `has_jump_markup / jump_fragments / is_note_like`: note and hyperlink
  metadata;
- `alignment_role`: `align` or `retain`;
- `paratext_kind`: `body / toc / note_body / chapter_heading / frontmatter / backmatter / metadata / unknown`;
- `filter_reason`: heuristic classification reason.

### `AlignmentResult`

`AlignmentResult` is the stable intermediate layer between extraction and the
builder.

Important fields:

- `pairs`: aligned `AlignedPair[]`;
- `source_lang / target_lang`;
- `granularity`;
- `extract_mode`;
- `retained_source_segments`;
- `retained_target_segments`.

In addition to aligned content, it stores one-sided pairs for later review and
serves as the builder-only debugging input so the model does not need to be
rerun.

## 4. EPUB extraction

Relevant modules:

- `bookalign/epub/reader.py`
- `bookalign/epub/tag_filters.py`
- `bookalign/epub/extractor.py`
- `bookalign/epub/cfi.py`
- `bookalign/epub/sentence_splitter.py`

The current extraction strategy is `filtered_preserve`.

It does not send every visible text fragment to the aligner. It first
classifies content:

- body text enters `alignment_segments`;
- TOC entries, notes, chapter headings, and front/back matter enter
  `retained_segments`.

Extraction also:

- generates paragraph- and sentence-level CFIs;
- preserves original HTML so the builder can restore structure where possible;
- records footnote references, backlinks, and anchor metadata; and
- stores sentence ranges within paragraphs for inline reconstruction.

## 5. Chapter matching is no longer the only anchor

This is one of the most important changes in the current workflow.

Previously, chapter matching could be viewed as:

```text
extract chapters -> match chapters -> align sentences
```

Now `chapter_id` must not be treated as an absolute anchor.

Real problems include:

- the chapter shown by `list_book_chapters(...)` may not match the ownership of
  sentence records;
- one `chapter_id` may contain multiple body regions;
- paragraph indexes may reset within a visible chapter bucket; and
- front matter, chronology, or note blocks may be mixed into body segments.

The recommended process is:

1. Inspect `list_book_chapters(...)`.
2. Inspect `get_chapter_preview(...)`.
3. Inspect `get_chapter_structure(...)`.
4. Sample `sentence_segments`.
5. Add chapter information to the formal slice plan only when these views
   agree.

Chapter matching is therefore a candidate-suggestion layer, not the final
execution layer.

## 6. Alignment layer

Body alignment still uses the Bertalign path, wrapped in:

- `bookalign/align/bertalign_adapter.py`
- `bookalign/align/aligner.py`

Its core capabilities are:

- multilingual sentence embeddings;
- embedding similarity; and
- dynamic-programming alignment.

It supports:

- `1-1`
- `1-N`
- `N-1`
- `N-M`
- `1-0 / 0-1`

This is more reliable than retrieving the single most similar sentence,
especially for literary translations with splits, merges, additions, and
omissions.

The main engineering change is the execution boundary:

- whole-book implicit pairing is no longer the default;
- production requires an explicit `slice_plan`; and
- unmatched segments must be reviewed after each alignment round.

## 7. Why JSON and review artifacts are retained

Alignment is expensive, human judgment often needs to be repeated, and builder
changes are frequent. Persisting intermediate artifacts is therefore part of
the current design, not an optional convenience.

Artifacts make it possible to:

- change styles or builder behavior without rerunning the model;
- inspect suspicious alignment windows independently;
- separate alignment problems from reconstruction problems;
- review unmatched regions separately; and
- create a review checkpoint before building.

Important artifacts include:

- `source_extraction.json`
- `target_extraction.json`
- `slice_manifest.json`
- `alignment.json`
- `alignment_report.json`
- `review.html`

## 8. Builder design

Relevant modules:

- `bookalign/epub/builder.py`
- `bookalign/pipeline.py`

There are two main output strategies:

- `simple`: generate a new block-oriented bilingual EPUB;
- `source_layout`: write the translation back into the original EPUB
  structure.

`source_layout` is preferred for public use because it produces a more natural
reading experience and better matches the project's goal.

### `paragraph` mode

Writes the translation after each source paragraph.

Advantages:

- more robust;
- less destructive to the source EPUB structure; and
- better reader compatibility.

Disadvantage:

- translation granularity is coarser.

### `inline` mode

Rewrites each source block with interleaved source and target sentences.

Advantages:

- the closest source/translation pairing;
- better for language learning and close reading.

Disadvantages:

- depends more heavily on sentence and paragraph position data; and
- is more sensitive to dirty EPUB styles, nested tags, and unusual line breaks.

### Current builder default

Chinese translation paragraphs receive two leading spaces during build. This
is an explicit reading-layout choice rather than only a CSS visual indent.

## 9. Notes, retained content, and unmatched segments

This is the builder's most difficult layer and a major reason for the
skill-first workflow.

Current rules:

- note bodies do not participate in body alignment;
- retained content remains in JSON;
- the builder writes notes and other retained content to separate XHTML
  documents;
- note references in the body are rewritten as clickable footnote markers; and
- backlinks in note pages point to anchors in the rebuilt body where possible.

After an alignment round, do not inspect only the summary. Inspect:

- source-only pairs;
- target-only pairs; and
- consecutive unmatched regions.

This is why APIs such as `review_unaligned_segments(...)` exist.

## 10. Current limitations

### EPUB health has a large effect

Real EPUBs are often messy. Common problems include:

- empty TOC links;
- inconsistent note DOM structures;
- chapters split into many spans;
- paragraphs and line breaks used inconsistently; and
- introductions, appendices, and metadata mixed into body content.

Much of the engineering effort therefore goes into input compatibility rather
than alignment-algorithm optimization.

### Language coverage is still limited

The most stable combinations are:

- Japanese original to Chinese translation;
- English original to Chinese translation.

The Spanish path works, but is less stable than the two combinations above.

### Literary text is not regular parallel data

Literary translations commonly include:

- sentence splitting;
- sentence merging;
- inversion;
- explanatory additions; and
- rhetorical substitutions.

Sentence-level alignment is therefore not an absolute ground truth. It
approximates a readable result.

### The CLI whole-book pipeline is still optimistic

Although the CLI continues to work, it should not be the production default.
If a book has chapter drift, mixed content, commentary blocks, or index resets,
one-shot whole-book execution remains risky.

## 11. Future direction

### Continue strengthening staged production

The most valuable next improvements are:

- more reliable drift and mixed-content preflight checks;
- clearer slice-planning capabilities;
- automatic anomaly scans before building; and
- better organization of review artifacts.

### Improve reader compatibility

Potential improvements include:

- finer indentation and paragraph-spacing controls;
- consistent note styling;
- dark-mode and additional reader compatibility testing; and
- dedicated handling for images, captions, and poetry.

### Move toward a reader component

The longer-term direction is a reader-integrated parallel-reading experience
rather than only an offline EPUB export.

The repository has already shown that official translation plus original text,
automatic alignment, and EPUB reconstruction are viable. The next natural
form is a parallel-reading component inside a reader.
