# HashedContext

**Scaling GPU KV-Cache Efficiency and Inference Concurrency for Autonomous Agents: A Dual-Mode Architecture**

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.23698048.svg)](https://doi.org/10.5281/zenodo.23698048)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](LICENSE)
![Status: preprint v0.3](https://img.shields.io/badge/status-preprint%20v0.3-orange.svg)

> **Preprint, not peer-reviewed.** v0.3 is a major revision. It corrects several numbers and formulas from v0.1/v0.2, adds a second prompt layout (the dual-mode architecture), and replaces analytical projections with simulated measurements and a first live-LLM test. See [What's new in v0.3](#whats-new-in-v03).

**Author:** Rahat Hasan · [Akkhar-Labs](https://github.com/Akkhar-Labs) · Bogura, Bangladesh · ORCID [0009-0003-6184-5269](https://orcid.org/0009-0003-6184-5269)

| Version | Date | PDF | Archive |
|---|---|---|---|
| **v0.3** (latest) | 2026-10-29 | [`paper/HashedContext-draft-v0.3.pdf`](paper/HashedContext-draft-v0.3.pdf) | Not yet archived on Zenodo |
| v0.2 | 2026-10-27 | [`paper/HashedContext-draft-v0.2.pdf`](paper/HashedContext-draft-v0.2.pdf) | Zenodo · [10.5281/zenodo.23698048](https://doi.org/10.5281/zenodo.23698048) |
| v0.1 | 2026-10-26 | [`paper/HashedContext-draft-v0.1.pdf`](paper/HashedContext-draft-v0.1.pdf) | Zenodo · [10.5281/zenodo.23668958](https://doi.org/10.5281/zenodo.23668958) |

The v0.1 and v0.2 PDFs in this repository are identical to the files archived on Zenodo. The concept DOI [10.5281/zenodo.23668957](https://doi.org/10.5281/zenodo.23668957) always resolves to the newest archived version.

![HashedContext overview: append-only context vs. HashedContext, the HashGate operation, and the serving stack with the local HashVault](assets/overview.png)

*Overview figure from v0.2. (a) Append-only agents keep one full copy of a file per turn. (b) HashedContext replaces historical copies with SHA-256 stubs and keeps one live copy at the prompt tail. (c) Historical bytes live in a local content-addressed vault and are resolved on demand. In v0.3 the stub size is corrected to ~67 tokens, from the ~35 shown here.*

---

## The problem

Autonomous coding agents (Claude Code, Cursor, Cline, Devin, …) re-send the same unchanged source files into the conversation on almost every turn. With `M` files of `N` tokens over `P` turns, each prompt grows as `O(P × M × N)` and the cumulative prompt load as `O(P² × M × N)`. That wastes GPU KV-cache memory, slows prefill and decoding, and limits how many sessions an inference server can serve at once.

## The idea

HashedContext is a **content-addressed context architecture** for multi-turn agents. It treats conversation history as a *state-transition log*, not a filesystem archive. Every file observation is SHA-256 hashed. Past copies of a file are replaced by a compact `[HASH_STUB]` (~67 tokens). Every version is kept in a local content-addressed vault and can be restored losslessly by its hash.

```
[HASH_STUB]
path: src/auth/session.ts
sha256: e4b2c91f…8741aa
state: CACHED
[/HASH_STUB]
```

### Dual-mode architecture

The best prompt layout depends on whether the serving stack keeps a warm prefix cache, so HashedContext offers two lossless modes:

- **Mode A: Authoritative Tail.** Every historical file copy becomes a stub, and one complete, self-attesting copy of each current file sits at the end of the prompt. Uses the least KV memory.
- **Mode B: Lazy Tail.** History is never rewritten. Each file is sent in full once, unchanged re-reads become stubs, and changes are sent as line diffs. After `max_diffs_before_rebase` diffs (default 10) the full file is sent again. Because each prompt is a strict prefix of the next, a prefix cache can reuse almost all of it.

Both modes can run **client-side** (IDE extensions, agent CLIs) or **server-side** as a gateway filter in front of engines like vLLM or SGLang, with no changes to model weights or the inference engine.

## Results (v0.3)

Measured with a reference implementation driven by 20,000 scripted coding agents on four real open-source repositories (824,728 turns, exact BPE token counts), and a GPU serving simulator using H100 constants. GPU numbers come from simulation, not physical hardware.

| | Mode A: Authoritative Tail | Mode B: Lazy Tail (N=10) |
|---|---:|---:|
| KV-cache per request vs append-only | **−88.7%** | −84.8% |
| KV-cache vs append-only + compaction | **−69.0%** | −58.2% |
| Prompt tokens computed, warm prefix cache | +3.6% | **−88.1%** |
| Prompt tokens computed, no prefix cache | **−88.7%** | −84.8% |
| 16-GPU cluster, 512 agents: turns/hour vs baseline | **3.57×** | 3.21× |
| Finds an external one-line change without vault lookup | file only | file, line and text |

Every simulated session reconstructed byte-exact under both modes.

**Which mode to use**

| Scenario | Recommended |
|---|---|
| Prefix cache on, light load (1–8 agents per GPU) | **Mode B** (1.07–1.76× the turns of Mode A) |
| Prefix cache on, heavy load (16+ agents per GPU) | **Mode A** (Mode B: 0.80–0.91×) |
| No prefix cache, or requests not routed to the same cache | **Mode A** (Mode B: 0.66–0.91×) |
| Billed API with discounted cached input tokens | **Mode B** |
| Lowest KV memory per request | **Mode A** |
| Long edit loops, accuracy-critical work | **Mode A**, or **Mode B with N=5** |

**Live-LLM diff-chain test.** Mode B's main risk is that the model has to rebuild a file from its last full copy plus a chain of diffs ("lost in the middle"). We tested this on `deepseek-v4.1-flash` (30 cases; a file with 10 edits; the model is asked for the line that follows a given line):

| Prompt | Diffs to apply | Correct |
|---|---:|---:|
| Append-only | 0 | 29/30 (96.7%) |
| Mode A | 0 | 28/30 (93.3%) |
| Mode B, N=10 | 10 | 27/30 (90.0%) |
| Mode B, N=5 | 4 | 28/30 (93.3%) |

The differences are one or two cases, within the noise of 30 samples on one model. For accuracy-critical work we recommend `max_diffs_before_rebase = 5`. Larger, multi-model tests are planned.

**Pilot session.** A single 5-turn session on DeepSeek V4 (Mode A) cut prompt tokens by 57.7% at turn 4. It is one sample and illustrates the mechanism only.

## What's new in v0.3

**New**
- **Dual-mode architecture.** Mode B (Lazy Tail) with line diffs and a bounded diff chain (`max_diffs_before_rebase`), a new algorithm, and scenario-based recommendations for choosing a mode.
- **Simulated validation** replaces the analytical projections: a 20,000-agent fleet, a GPU serving simulator, a real HTTP gateway load test, and mechanical needle tests.
- **First live-LLM test** of Mode B diff chains.

**Corrections to v0.1/v0.2**
- Stub size is ~67 tokens, not ~35. SHA-256 costs ~1.8 ms/MB, not 2–5 µs/MB. Both are still negligible next to prefill.
- Cost is still quadratic in the number of turns. HashedContext shrinks the coefficient by ~25–30× but does not make it linear, and the cost ratio tends to a finite constant, not infinity. The budget-limited loop-depth formula is replaced.
- Append-only prompts already achieve high prefix-cache hit rates. Mode A's benefit is lower KV residency, not higher cache reuse.
- Decode speedup counts model-weight reads: 1.39× per request at batch size 1, with the larger gain from bigger batches.
- In gateway-native mode the gateway still parses the full payload. Network and parsing savings require edge-native mode.
- The pilot telemetry table drops the extrapolated "normalized turn 5" row and the analytical "file-specific tokens" row.
- Signal-to-noise, Amdahl and attention-entropy statements are corrected or labelled as hypotheses.

## Status

- [x] Architecture and formal model
- [x] Early benchmark on a production endpoint
- [x] Preprint v0.1 archived on Zenodo with a DOI
- [x] Preprint v0.2 archived on Zenodo, adding figures for architecture, memory, caching, and cost analysis
- [x] Reference implementation and simulation harness (fleet, GPU serving simulator, gateway load test)
- [x] Dual-mode architecture (Mode A / Mode B) with simulated trade-offs
- [x] First live-LLM test of Mode B diff chains
- [ ] Preprint v0.3 archived on Zenodo
- [ ] Larger live-LLM evaluation across models and tasks
- [ ] Measurements on physical GPUs
- [ ] Submission to arXiv / a peer-reviewed venue

## Related work by the author

- [ZRCP](https://github.com/sover-labs/zrcp): hash-based protocol specification for eliminating redundant context between AI agents
- [Brud Code](https://github.com/rahatarch/brud): deterministic paste-and-apply execution engine for AI-assisted coding

## Citation

If you reference this work, please cite the latest archived version, currently v0.2 (see also [`CITATION.cff`](CITATION.cff)):

```bibtex
@misc{hasan2026hashedcontext,
  title     = {Scaling GPU KV-Cache Efficiency and Inference Concurrency for Autonomous Agents with HashedContext},
  author    = {Hasan, Rahat},
  year      = {2026},
  publisher = {Zenodo},
  version   = {v0.2},
  doi       = {10.5281/zenodo.23698048},
  url       = {https://doi.org/10.5281/zenodo.23698048}
}
```

To cite the work independent of version, use the concept DOI [10.5281/zenodo.23668957](https://doi.org/10.5281/zenodo.23668957). Once v0.3 is archived on Zenodo, this section will point to its DOI.

## License

The paper is licensed under [CC BY 4.0](LICENSE). You may share and adapt it with attribution.

## Contact

Rahat Hasan · [rahathasan5773@gmail.com](mailto:rahathasan5773@gmail.com) · [github.com/rahatarch](https://github.com/rahatarch)
