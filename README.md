<!--
Keywords: code search, semantic code search, AI agent, LLM, BM25, embeddings,
          tree-sitter, AST, dependency graph, impact analysis, Rust, CLI,
          Claude Code, Codex, Cursor, grep replacement, token reduction,
          potion-code, model2vec, hybrid search, RRF, build output digest,
          CI log compression, korean code search, 한글 코드 검색
-->

# semble_rs

> **Fast, AI-agent-native code search + build/test/CI output compression — written in Rust.**
> One hybrid (BM25 + semantic) search replaces `grep`/`cat`/`read`/`ls`;
> `semble_rs digest` collapses 3 MB CI logs into 35 KB.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Rust](https://img.shields.io/badge/rust-1.75%2B-orange.svg)](https://www.rust-lang.org)
[![Platform](https://img.shields.io/badge/platform-macOS%20%7C%20Linux%20%7C%20Windows-blue.svg)](#installation)
[![Token savings](https://img.shields.io/badge/agent%20tokens-up%20to%20--98.9%25-brightgreen.svg)](#digest--buildtestci-output)
[![한국어](https://img.shields.io/badge/한국어-README.ko.md-blue.svg)](./README.ko.md)

**Keywords:** AI code search · LLM agent tools · grep/cat replacement · BM25 + embeddings · Tree-sitter AST · build / CI log compression · Rust CLI · Claude Code · Codex · Cursor · 한글 코드 검색

한국어 사용자는 [README.ko.md](./README.ko.md)를 참고하세요.

---

## What it does

AI agents burn tokens two ways:
1. **Exploring code** — repeated `grep` → `cat` → `read` reads megabytes of irrelevant content.
2. **Reading build / CI output** — `cargo build`, `pnpm install`, `gh run view --log` dump tens of KB to MB of progress noise.

`semble_rs` collapses both:

| Stage | Without `semble_rs` | With `semble_rs` | Savings |
|---|---|---|---|
| Code lookup | `ls` → `grep` → `cat file₁` → `cat file₂` → … | `semble_rs search "auth flow" . --outline` | **-93%** typical session |
| CI failure debug | `gh run view <id> --log-failed` (3.3 MB raw) | `gh run view … \| semble_rs digest` (35 KB) | **-98.9%** |

It is a single Rust binary, no runtime dependencies, with a Rust rewrite of [MinishLab/semble](https://github.com/MinishLab/semble) at its core plus dependency graphs, AST chunking, Korean/CJK Unicode search, and an output-digest pipeline.

## Installation

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
git clone https://github.com/johunsang/semble_rs.git && cd semble_rs
cargo install --path .
```

The binary lands at `~/.cargo/bin/semble_rs`. On first run, the default embedding model `minishlab/potion-code-16M` (~60 MB) is downloaded from HuggingFace.

## Quick start

```bash
# Code search (replaces grep / cat / read / ls)
semble_rs search "auth flow" ./my-project --outline      # pass 1: structural overview
semble_rs search "loginWithEmail" ./my-project --compact # pass 2: matching lines

# Dependencies and change-impact
semble_rs deps   src/lib/auth.ts ./my-project
semble_rs impact src/lib/auth.ts ./my-project

# Build / test / CI output compression
cargo build 2>&1     | semble_rs digest
pnpm install 2>&1    | semble_rs digest
gh run view <id> --log-failed | semble_rs digest
```

## Search — output modes

| Mode | Output | Token cost vs `--compact` | When to use |
|---|---|---|---|
| `--outline` | One signature line per chunk | **-47%** | First-pass structural scan |
| `--group` | Directory grouping + match lines capped at 3 (`+N` overflow) | -47% | Many match lines per chunk |
| `--compact` | Score + path + every matching line | baseline | Precision scan |
| `--json --strip` | Chunk bodies (comments stripped) | +800% | Tooling / pipeline integration |
| `--json` | Chunk bodies (raw) | +900% | Tooling / pipeline integration |

**Recommended:** `--outline` to overview → `--compact` to narrow → `--json --strip` only if the chunk body itself is needed.

`--outline` accuracy on the 33-query self-benchmark: **100% well-formed** signatures (parens balanced, no truncation).

## `digest` — build / test / CI output

Auto-detects and compresses output from common toolchains. Errors and failures are never lost — only progress lines collapse to counts.

**Supported handlers:** `cargo`, `pnpm`/`npm`/`yarn`/`bun`, `tsc`, `pytest`, `go test`, `gradle`, `ruff`, `mypy`, `clang`/`gcc`/`cmake`/`make`/`swiftc` (`compiler`), GitHub Actions (`ci`). Unknown formats pass through unchanged.

Measurements on 15 real-world fixtures:

| Fixture | Raw → digest | Savings |
|---|---|---|
| `cargo build` (clean, 218 crates) | 7,611 B → 59 B | **-99.2%** |
| `cargo test` (45 passing) | 3,368 B → 369 B | -89.0% |
| `pnpm install` | 1,323 B → 349 B | -73.6% |
| `tsc` (13 errors, 5 codes) | 1,085 B → 648 B | -40.3% |
| `pytest` (4 failures) | 2,762 B → 2,330 B | -15.6% |
| **GitHub Actions log (rust-lang/rust failed CI, real)** | **3.3 MB → 35 KB** | **-98.9%** ⭐ |
| `go test` (with panic + stack) | 1,034 B → 475 B | -54.1% |
| `gradle test` (2 failures) | 1,232 B → 522 B | -57.6% |
| `ruff` (9 violations, 3 codes) | 624 B → 597 B | -4.3% |
| `mypy` | 336 B → 237 B | -29.5% |
| `clang`/`cmake`/`swift` compilers | ~600 B (progress stripped) | -3 ~ -30% |
| **TOTAL (15 fixtures)** | **3.33 MB → 43 KB** | **-98.7%** |

```bash
# Force a specific handler when auto-detect misses
semble_rs digest --format ci  ci_log.txt
semble_rs digest --format gradle gradle_test.log

# Inspect which handler was picked
semble_rs digest --show-format my_log.txt
```

**Preservation guarantees**
- File:line:col, traceback, panic stack, failed-step bodies are always kept.
- Repeated error codes are grouped (e.g. `TS2322` × 9 → top 3 + `+6 more`).
- CI `##[group]` blocks: successful groups collapse to one line; failed groups keep their trailing 80 lines verbatim.

## Supported languages (search)

| Language | Search | AST chunking | Dependency graph |
| --- | --- | --- | --- |
| Rust | ✓ | ✓ | ✓ |
| Python | ✓ | ✓ | ✓ |
| JavaScript | ✓ | ✓ | ✓ |
| TypeScript | ✓ | ✓ | ✓ |
| Go | ✓ | ✓ | ✓ |
| Java | ✓ | ✓ | ✓ |
| C / C++ | ✓ | ✓ | ✓ |
| **Kotlin** (v0.3.0+) | ✓ | ✓ | ✓ |
| Ruby, PHP, Swift, others | ✓ | line-based fallback | — |

## Search quality

Default embedding model (`potion-code-16M`) on a 50-query self-benchmark:

| Metric | Score |
|---|---|
| Recall@1 | 70% |
| Recall@5 | 96% |
| **Recall@10** | **100%** |
| MRR | 0.81 |
| Korean Recall@5 | 60% |

The benchmark / eval scripts live in [`semble-train/`](./semble-train) (Python).

### Experimental: swap the embedding model

Point `SEMBLE_MODEL_PATH` at a local [model2vec](https://github.com/MinishLab/model2vec) output directory (`tokenizer.json` + `model.safetensors`):

```bash
SEMBLE_MODEL_PATH=/path/to/my-distilled-model semble_rs search "query" ./my-project --compact
```

We tried distilling `SFR-Embedding-Code-400M_R` — it underperformed our default (R@10 96% vs 100%, Korean R@5 40% vs 60%) because its vocabulary is English-code-only and breaks the Korean BM25 + semantic synergy. Pick teachers whose vocab covers your real corpus.

## Integration with AI coding agents

### Global `CLAUDE.md` (Claude Code) and `AGENTS.md` (Codex)

Drop a section like the following into `~/.claude/CLAUDE.md` and `~/.codex/AGENTS.md`:

```markdown
# semble_rs — replaces grep, cat, read, ls + compresses build/CI output

ALWAYS use these instead of raw grep/cat/find/read:

  semble_rs search "<feature>" /path --outline      # 1단계 탐색
  semble_rs search "<symbol>"  /path --compact      # 2단계 정밀 탐색
  semble_rs deps   <file>      /path
  semble_rs impact <file>      /path

ALWAYS pipe build / test / CI output through `semble_rs digest`:

  cargo build 2>&1   | semble_rs digest
  pnpm install 2>&1  | semble_rs digest
  pytest 2>&1        | semble_rs digest
  gradle test 2>&1   | semble_rs digest
  gh run view <id> --log-failed | semble_rs digest

Rules: never guess symbol names (use natural-language descriptions instead),
always pass a directory path (not a file path), and only fall back to `grep`
when semble_rs results are insufficient.
```

### Per-project (any agent)

Same content in a project-root `CLAUDE.md` or `AGENTS.md` works for Claude Code, Codex, Cursor (`.cursorrules`), Aider, and OpenHands.

## License

MIT — see [LICENSE](./LICENSE).

Credits:
- Upstream: [MinishLab/semble](https://github.com/MinishLab/semble) — the Python implementation this Rust port draws from.
- Embedding model: [`minishlab/potion-code-16M`](https://huggingface.co/minishlab/potion-code-16M).
- Built on: `tree-sitter`, `ndarray`, `safetensors`, `tokenizers`, `hf-hub`, `ignore`, `clap`.
