# Chrome Classifier Study

Focused study of Chromium's local `PASSAGE_EMBEDDER`, `EDU_CLASSIFIER`, and
`SHOPPING_CLASSIFIER` models, with emphasis on what is verified, what the
models can ingest, and how they behave in a local research pipeline.

## Model source

The model files used by this study are published at
[`dejanseo/chrome_models`](https://huggingface.co/dejanseo/chrome_models) on
Hugging Face.

This repository isolates the classifier study as a standalone research
artifact.

## Executive summary

- `PASSAGE_EMBEDDER` is a DualEncoder that produces a 768-dimensional,
  L2-normalized embedding. Cosine similarity is therefore the intended
  comparison metric.
- `EDU_CLASSIFIER` is a linear probe over the passage embedding. Its scores
  are currently treated as **unverified** because its metadata does not expose
  an embedder-version compatibility field.
- `SHOPPING_CLASSIFIER` consumes up to 10 passages and uses mean pooling. Its
  required embedder version matches the `PASSAGE_EMBEDDER` build tested.
- The shared embedder can produce similarity, education, and shopping signals
  in one local embedding pass.
- The hard 64-token input window makes full-page classification impossible
  without chunk selection. A 1,877-word page measured 3,687 tokens / 58 chunks;
  Shopping's 10-passage ceiling covers only 640 tokens, roughly 17% of that
  page.

## The three source findings

### 1. Mechanism verification — `9237c2f`

Source commit: `9237c2f`

Verified from Chromium's model graphs, decoded metadata, and the corresponding
Chromium protobuf schemas rather than inferred from a reference implementation:

| Model | Verified behavior | Confidence / caveat |
| --- | --- | --- |
| `PASSAGE_EMBEDDER` | 8-layer pre-norm Transformer; max pooling; L2-normalized 768-d output; 64-token input window | Mechanism verified from the TFLite graph and metadata |
| `EDU_CLASSIFIER` | Single logistic-regression layer (`MatMul` + `BiasAdd`) over the 768-d embedding | No classifier metadata block or embedder-version field; scores remain unverified |
| `SHOPPING_CLASSIFIER` | `max_passages=10`; `pooling_strategy=MEAN`; compatible embedder version | Metadata and required embedder version matched the tested embedder |

The EDU model timestamp was approximately seven months older than the
embedder build in use. Without a compatibility field, that possible
train/inference mismatch cannot be ruled out from metadata alone.

### 2. Ingestion capacity — `8b3646f`

Source commit: `8b3646f`

The 64-token input window is a hard per-call limit. The local truncation path
is silent, so callers must measure or select text before inference.

Observed measurements:

- Title + metadata for 50 sources: 32–63 tokens each, average approximately
  45, with no truncation.
- One real page (`kivo.io/dms`): 1,877 words, 3,687 tokens, 58 chunks.
- `SHOPPING_CLASSIFIER`'s 10-passage limit: 640 tokens, or about 17% of that
  page.
- `EDU_CLASSIFIER`: no confirmed multi-chunk aggregation behavior.

Implication: a production full-page path needs a chunk-selection or
prioritization stage. Feeding every chunk directly into Shopping is not
supported by the model's own capacity, and any EDU aggregation strategy would
be a new design rather than a reproduced Chromium behavior.

### 3. Local integration — `3213820`

Source commit: `3213820`

The models were integrated locally before any remote deployment work:

```text
source -> PASSAGE_EMBEDDER ->
  cosine similarity
  EDU_CLASSIFIER score
  SHOPPING_CLASSIFIER score
```

Results from the same query and sources used in the earlier Jina-based test:

- Veeva's homepage improved from `0.541` with Jina similarity to `0.872`
  with the local `PASSAGE_EMBEDDER` (rank 30/50).
- EDU scores ranged from `0.6` to `0.98` across the result set and did not
  discriminate this particular query.
- `SHOPPING_CLASSIFIER` correctly isolated ComplianceQuest as a vendor-pitch
  outlier with a score of `0.956`; the other results were near zero.

The practical value is one local embedding pass producing three signals,
without a separate Jina similarity request.

## Combined EDU + SHOP discovery

The key study discovery is that the EDU and Shopping classifiers work best as
a combined signal rather than as independent filters.

### What the classifiers measure

Both are Chrome on-device models that run locally and without per-request API
cost:

- **EDU classifier** — scores how educational or formal a snippet is, from 0
  to 1.
- **Sales/Shop classifier** — Chrome's Shopping model, scoring how commercial
  or e-commerce-oriented a snippet is.

### Methodology correction

A naive whole-dataset correlation between the two signals showed almost no
relationship: `corr(1 - edu, shop) = 0.135`. That result was misleading because
Shopping was approximately zero for 47 of 50 rows. The correct test was to
compare EDU on rows where Shopping actually fired against the remaining rows.

### Discovery

Rows with a real Shopping signal—Arena QMS (`shop=1.00`), Veeva
(`shop=0.301`), and ComplianceQuest (`shop=0.956`)—had measurably lower EDU,
around `0.77`, than the other rows, around `0.87–0.90`.

This supports an inverse-pair interpretation:

> high SHOP + low EDU = commercial intent

The combined rule is therefore **low EDU and high SHOP**, not high SHOP alone.
The commercial complement should be carried explicitly as `inv_edu = 1 - edu`
alongside `shop`.

### Recommended pre-screen rule

| Condition | Action |
| --- | --- |
| `edu >= 0.7` | Keep as clearly educational |
| `0.6 <= edu < 0.7` | Marginal; let the reranker decide |
| `edu < 0.6` and/or SHOP fires high | Reject or classify as commercial/service/job content |

Designate a candidate as commercial using the stricter inverse-pair rule:
`shop >= ~0.3 AND edu < ~0.6`.

### Intended workflow

1. Search using title and snippet only. Do not fetch full pages at this stage.
2. Score each candidate from one local `PASSAGE_EMBEDDER` call:
   `sim` (query relevance), `edu` (educational content), and `shop`
   (commercial signal).
3. Pre-screen with the EDU + SHOP inverse pair: keep educational candidates,
   drop clear commercial candidates, and pass marginal cases onward.
4. Fetch survivors and split their pages into header sections.
5. Rerank sections by relevance using Jina or a local reranker.
6. Save the best sections to the RAG Knowledge collection.
7. Synthesize a grounded answer, then apply the voice and scoring stages.

### Caveats

1. **SHOP is narrowly tuned.** It fires on literal e-commerce vocabulary such
   as cart, checkout, free shipping, and in-stock. It is not a general
   commercial-page detector: it missed a commercial car-rental page
   (`shop=0.000`) and false-positived a regulatory document
   (`shop=0.759`) because of catalog-style formatting. Read it with EDU.
2. **`inv_edu` is the stronger complement for professional and regulatory
   content.** Low EDU often identifies service pages, CROs, and job boards even
   when SHOP does not fire.
3. **Scale matters.** Use title and snippet text, not full pages. Shopping
   covers only about 640 tokens—roughly 17% of a typical page—and is sensitive
   to word order: shuffling text collapsed one observed score from `0.980` to
   `0.009`.
4. **This is a pre-screen, not a final gate.** Reranking makes the final
   selection; the EDU + SHOP pair cheaply thins the candidate pool.

## Study status

| Area | Status |
| --- | --- |
| Embedder architecture and metric | Verified |
| Shopping pooling and version compatibility | Verified for tested build |
| EDU score interpretation | Open; treat as unverified |
| EDU + SHOP commercial signal | Promising inverse-pair pre-screen; not a final gate |
| Short title/metadata ingestion | Measured and workable |
| Full-page ingestion | Capacity constraint measured |
| Chunk selection | Not yet implemented |
| Remote deployment | Deliberately out of scope for this breakout |

## Next experiments

1. Build a reproducible chunk-selection benchmark for full-page inputs.
2. Compare title/meta, lead-section, and selected-content strategies against
   human relevance labels.
3. Test whether EDU scores are stable across compatible and intentionally
   mismatched embedder versions.
4. Evaluate whether Shopping remains useful after chunk selection rather than
   metadata-only input.
5. Record model hashes, tokenizer versions, and exact input text for every
   experiment.

## Provenance

The model artifacts are sourced from
[`dejanseo/chrome_models`](https://huggingface.co/dejanseo/chrome_models).
The three source findings are preserved by commit identifiers `9237c2f`,
`8b3646f`, and `3213820`. This breakout repository is intended to isolate the
classifier study from the broader persona and Mercury Researcher work.
