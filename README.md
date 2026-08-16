# Chrome Classifier Study

Focused study of Chromium's local `PASSAGE_EMBEDDER`, `EDU_CLASSIFIER`, and
`SHOPPING_CLASSIFIER` models, with emphasis on what is verified, what the
models can ingest, and how they behave in a local research pipeline.

## Model source

The model files used by this study are published at
[`dejanseo/chrome_models`](https://huggingface.co/dejanseo/chrome_models) on
Hugging Face.

This repository is a breakout of the study recorded in
[openwebui-writer-personas PR #17](https://github.com/blazingbunny/openwebui-writer-personas/pull/17).

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

[Original commit](https://github.com/blazingbunny/openwebui-writer-personas/commit/9237c2ff69cd444600d21efc8a825d8618ac5d78)

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

[Original commit](https://github.com/blazingbunny/openwebui-writer-personas/commit/8b3646fdc355bacb45237fca7ba26f4ea3246912)

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

[Original commit](https://github.com/blazingbunny/openwebui-writer-personas/commit/3213820f72dd86463fd7f78614b8b138503ff6fe)

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

## Study status

| Area | Status |
| --- | --- |
| Embedder architecture and metric | Verified |
| Shopping pooling and version compatibility | Verified for tested build |
| EDU score interpretation | Open; treat as unverified |
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
All three findings originated in the
[`openwebui-writer-personas`](https://github.com/blazingbunny/openwebui-writer-personas)
repository and were documented in PR #17. This breakout repository is intended
to isolate the classifier study from the broader persona and Mercury Researcher
work.
