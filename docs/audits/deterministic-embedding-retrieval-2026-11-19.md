## Verdict
**The published v0.1 artifacts are internally consistent and independently checkable. They support a bounded smoke-test result, not universal determinism, semantic retrieval accuracy, or completion of the planned 10,000-run study.**

Reviewed 2026-11-19. Dataset revision: `2de315744e118112187bab768a1859b239c49cf8`.
[Dataset and source artifacts](https://huggingface.co/datasets/igorholt/deterministic-embedding-retrieval-proof/tree/2de315744e118112187bab768a1859b239c49cf8).
Engine attribution: upstream [lmtlssss/speedrunpapi](https://github.com/lmtlssss/speedrunpapi); dataset specifies this fork at `8479566cfd3b008f927184d32ac9b6a98deec3a5`.

### Independently recomputed from downloaded artifacts
| Check | Result |
|---|---|
| Embedding shape / dtype | 693 × 384, int16 |
| Pass 1 vs pass 2 | Array-equal |
| Permuted pass, aligned by chunk ID | 693/693 vectors equal |
| Canonical little-endian int16 array SHA-256 | `e3e0215c3f192fc8ae54c1fdb7cdb17d39966276cce9b528eef5880bdf0efb13` |
| Chunk IDs vs table | Exact order match |
| Unique normalized chunk texts | 693 |
| Reconstruct v2 by first-occurrence text dedup of v1 | Exact chunk-list match; 699 → 693 |
| Nonzero vector norms | 693/693 |
| Recomputed int64 dot matrix | Exact equality to published matrix |
| Recomputed float64 cosine matrix | Exact equality to published matrix |
| Seed 20261118 row permutation, recomputed and mapped back | Exact cosine equality |
| Self-query top-1 sets | 693/693 contain self; all singleton |
| Mean off-diagonal cosine | 0.16997688301673874 |
| Median / minimum self-retrieval margin | 0.33756615692099423 / 0.00015089711118621008 |
| Rows with margin below 0.001 | 4 |

These checks used NumPy 2.3.5 and independently written array calculations. No new model inference, PDF extraction, engine STS-B self-check, or 10,000-pass run was performed. The mean's final decimal differs from the report by approximately 2.8e-17 despite elementwise matrix equality: aggregate floating-point reductions are a separate reproducibility surface.

### Interpretation
- Self-query retrieval uses each stored embedding to query the same corpus containing itself. For nonzero vectors, self-cosine is mathematically maximal. This checks identity, collisions and implementation consistency; it is not a semantic-quality benchmark.
- The recorded natural-language result is 10/20 intended-document top-1 hits, using hand-authored queries and author-selected targets. This was read from the published results, not re-embedded here. Publish it as descriptive only.
- A top-1 set is permutation-equivariant when scores are fixed and mapped back to stable IDs. Deduplication alone does not repair a tie-breaking rule. The six removed IDs exactly match DEDUP.md: 07-000, 07-012, 07-030, 07-051, 07-058, 07-068.
- Two stored matching passes and one stored shuffled pass cannot establish every future run or every hardware/backend configuration. Even 10,000 successful runs would be finite empirical evidence, with scope determined by the tested environments.

### Findings and required corrections
1. **Fail-closed verification is missing.** `embed_chunks.py` prints equality only when exactly two passes are requested, never exits nonzero on inequality, and retains every pass in memory. At 10,000 passes the raw vectors alone require 5,322,240,000 bytes (~4.96 GiB), excluding overhead. Stream each pass, compare with a baseline, log hashes and metadata, and terminate nonzero on mismatch. `permutation_test.py` likewise writes FAIL without a failing exit code.
2. **The README's v2 rebuild is incomplete.** `chunk_pdfs.py` writes v1 only. The v2 commands consume an already-existing v2 table; no executable v1-to-v2 dedup step is supplied. Add that step and assert retained IDs, counts and text equivalence. PDF re-extraction also depends on original workspace files not included in this dataset; distinguish artifact verification from reconstruction from PDFs.
3. **Bootstrap is underspecified.** Document the pinned engine checkout, model/cache preparation and exact dependencies. `retrieve.py` imports the engine even for calculations that can use saved vectors alone. Provide a standalone artifact verifier.
4. **Tie handling remains inconsistent outside the headline metric.** The diagnostic described as “argmax-first” actually uses default `np.argsort`, which does not guarantee first-in-corpus tie ordering. NL top-5 also uses argsort. Define a set-valued result or stable-ID secondary key at every ranking boundary; do not equate set invariance with an invariant ordered list.
5. **Numerical guarantees need narrower wording.** Int64 dot accumulation is safe for 384 int16 components: absolute sums are bounded by 384 × 32768² = 412,316,860,416, well below int64 overflow and 2^53. The product of squared norms may exceed 2^53, so denominator arithmetic is not all integer-exact after conversion. Cross-platform float64 identity requires an explicit numerical contract and tests. Reject zero vectors and nonfinite scores, validate shapes/dtypes/IDs, and specify exact-tie semantics.
6. **The GitHub mirror link is not verified.** The README advertises `igor-holt/deterministic-embedding-retrieval-proof`; both authenticated GitHub file access and a public repository API request returned 404 during review. A 404 does not distinguish absence from restricted access. Restore the mirror or correct the link.
7. **Test depth.** Add exact duplicates without dedup, equal-direction vectors, near ties, zero vectors, changed batch sizes, independent processes, multiple seeds and multiple supported hosts. Freeze query judgments before a held-out semantic evaluation; report Recall@k/MRR separately from determinism.

### Website-ready wording
“Deterministic embedding retrieval: preliminary artifact verification. Corpus v2 contains 693 unique text chunks represented by 384-dimensional int16 vectors. The two published embedding passes match byte-for-byte; a published shuffled pass matches after alignment by chunk ID. Independent recomputation confirms the stored similarity matrix and 693/693 singleton self-retrieval sets. This is a smoke test, not a completed 10,000-run study or a semantic retrieval benchmark.”

### Acceptance gates
- [ ] Restore/correct the GitHub mirror link.
- [ ] Add a clean-environment bootstrap and executable v2 dedup step.
- [ ] Add streaming verification, nonzero failure exits and canonical serialization.
- [ ] Test ties, zero vectors, batch changes, independent processes and supported hardware.
- [ ] Publish the predeclared 10,000-run protocol and per-run evidence before upgrading claims.
- [ ] Keep empirical determinism and held-out semantic-quality results separate.

Sources: pinned dataset README.md, REPORT.md, DEDUP.md; chunk_table.json and chunk_table_v2.json; embeddings_pass1_v2.npz, embeddings_pass2_v2.npz, embeddings_perm_v2.npz; similarity_matrix_v2.npz; retrieval_results_v2.json; scripts/embed_chunks.py, retrieve.py, permutation_test.py, chunk_pdfs.py.
