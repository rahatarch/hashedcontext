# HashedContext

**Scaling GPU KV-Cache Efficiency and Inference Concurrency for Autonomous Agents**

> ⚠️ **Working draft (v0.1), not peer-reviewed.** Published here as a public, timestamped record of ongoing research. Content, math, and results may change.

**Author:** Rahat Hasan · [Akkhar-Labs](https://github.com/Akkhar-Labs) · Bogura, Bangladesh
**Draft date:** October 12, 2026

📄 **Read the draft:** [`paper/HashedContext-draft-v0.1.pdf`](paper/HashedContext-draft-v0.1.pdf)

---

## The problem

Autonomous coding agents (Claude Code, Cursor, Cline, Devin, …) re-send the same unchanged source files into the conversation on almost every turn. Over a long session, prompt size grows roughly as `O(M × N × P)` (files × tokens per file × turns). That wastes GPU KV-cache memory, slows prefill and decoding, and limits how many sessions an inference server can serve concurrently.

## The idea

HashedContext is a **content-addressed context architecture** for multi-turn agents. It treats conversation history as a *state-transition log*, not a filesystem archive:

1. **SHA-256 hash-gated file tracking:** every file observation is hashed before it is serialized.
2. **Historical deduplication via `[HASH_STUB]`:** past copies of a file are replaced by a compact (~35-token) stub.
3. **Attention-anchored authoritative tail:** exactly one live, full copy of each active file sits at the end of the prompt.
4. **On-demand resolution from a local HashVault:** any historical version can be restored losslessly by its hash when needed.

```
[HASH_STUB]
path: src/auth/session.ts
sha256: e4b2c91f…8741aa
state: CACHED
[/HASH_STUB]
```

It can run **client-side** (IDE extensions, agent CLIs) or **server-side** as a gateway filter in front of engines like vLLM or SGLang, with no changes to model weights or the inference engine.

## Early results

Initial measurements on DeepSeek V4 production endpoints (5-turn refactoring session on a ~4,000-token TypeScript file):

| Metric | Conventional | HashedContext | Reduction |
|---|---:|---:|---:|
| Prompt tokens (turn 4) | 27,747 | 11,746 | **57.7%** |
| Normalized turn 5 | 31,747 | 11,746 | **63.0%** |
| File-specific tokens | 20,000 | 4,140 | **79.3%** |

In this initial evaluation, the model still detected a single-line mutation in the authoritative tail with no observed loss in diff-tracking accuracy.

> Larger-scale benchmarks (more models, tasks, and session lengths) are in progress. Figures in the paper beyond the table above are analytical projections, not measurements.

## Status

- [x] Architecture and formal model
- [x] Early benchmark on a production endpoint
- [ ] Extended benchmarks across models and workloads
- [ ] Reference implementation
- [ ] Preprint submission

## Related work by the author

- [ZRCP](https://github.com/sover-labs/zrcp): hash-based protocol specification for eliminating redundant context between AI agents
- [Brud Code](https://github.com/rahatarch/brud): deterministic paste-and-apply execution engine for AI-assisted coding

## Citation

If you reference this draft, please cite it as below (see also [`CITATION.cff`](CITATION.cff)):

```bibtex
@misc{hasan2026hashedcontext,
  title  = {Scaling GPU KV-Cache Efficiency and Inference Concurrency for Autonomous Agents with HashedContext},
  author = {Hasan, Rahat},
  year   = {2026},
  note   = {Working draft v0.1},
  howpublished = {\url{https://github.com/rahatarch/hashedcontext}}
}
```

## License

The paper is licensed under [CC BY 4.0](LICENSE). You may share and adapt it with attribution.

## Contact

Rahat Hasan · [rahathasan5773@gmail.com](mailto:rahathasan5773@gmail.com) · [github.com/rahatarch](https://github.com/rahatarch)
