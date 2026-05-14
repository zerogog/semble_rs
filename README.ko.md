# semble_rs (한국어)

> **AI 에이전트용 코드 검색 + 빌드/테스트/CI 출력 압축 — Rust로 작성.**
> 하이브리드(BM25 + semantic) 검색으로 `grep`/`cat`/`read`/`ls`를 대체하고,
> `semble_rs digest`로 3 MB CI log를 35 KB로 압축합니다.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Rust](https://img.shields.io/badge/rust-1.75%2B-orange.svg)](https://www.rust-lang.org)
[![Token savings](https://img.shields.io/badge/agent%20tokens-up%20to%20--98.9%25-brightgreen.svg)](#digest--빌드--테스트--ci-출력-압축)
[![English](https://img.shields.io/badge/English-README.md-blue.svg)](./README.md)

---

## 무엇을 하나

AI 에이전트가 토큰을 폭발시키는 두 영역을 모두 잠급니다:

1. **코드 탐색** — `grep` → `cat` → `read` 반복하면 수 MB의 무관한 내용이 컨텍스트에 들어감
2. **빌드 / CI 출력 읽기** — `cargo build`, `pnpm install`, `gh run view --log`가 수십 KB~수 MB의 진행 라인을 쏟아냄

| 단계 | `semble_rs` 없이 | `semble_rs`로 | 절감 |
|---|---|---|---|
| 코드 찾기 | `ls` → `grep` → `cat 파일₁` → `cat 파일₂` → … | `semble_rs search "auth flow" . --outline` | **-93%** 평균 세션 |
| CI 실패 디버깅 | `gh run view <id> --log-failed` (3.3 MB raw) | `gh run view … \| semble_rs digest` (35 KB) | **-98.9%** |

단일 Rust 바이너리, 런타임 의존성 없음. [MinishLab/semble](https://github.com/MinishLab/semble)의 Rust 재작성 + 의존성 그래프 + AST 청킹 + 한글/CJK 유니코드 검색 + 출력 압축 파이프라인.

## 설치

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
git clone https://github.com/johunsang/semble_rs.git && cd semble_rs
cargo install --path .
```

`~/.cargo/bin/semble_rs`에 바이너리가 생성됩니다. 첫 실행 시 기본 임베딩 모델 `minishlab/potion-code-16M` (~60 MB)이 HuggingFace에서 다운로드됩니다.

## 빠른 시작

```bash
# 코드 검색 (grep / cat / read / ls 대체)
semble_rs search "인증 플로우" ./my-project --outline    # 1단계: 구조 파악
semble_rs search "loginWithEmail" ./my-project --compact # 2단계: 매칭 라인 확인

# 의존성 / 영향 분석
semble_rs deps   src/lib/auth.ts ./my-project
semble_rs impact src/lib/auth.ts ./my-project

# 빌드 / 테스트 / CI 출력 압축
cargo build 2>&1     | semble_rs digest
pnpm install 2>&1    | semble_rs digest
gh run view <id> --log-failed | semble_rs digest
```

## 검색 — 출력 모드

| 모드 | 출력 | 절감 (vs `--compact`) | 언제 사용 |
|---|---|---|---|
| `--outline` | 청크당 시그니처 1줄 | **-47%** | 1단계 구조 파악 |
| `--group` | 디렉토리 그룹핑 + 매칭 라인 최대 3개 (`+N`) | -47% | 매칭 라인 많을 때 |
| `--compact` | 점수 + 경로 + 모든 매칭 라인 | baseline | 정밀 탐색 |
| `--json --strip` | 청크 본문 (주석 제거) | +800% | 도구 통합 |
| `--json` | 청크 본문 (원본) | +900% | 도구 통합 |

**권장 워크플로:** `--outline` 개관 → `--compact` 좁히기 → 본문이 필요할 때만 `--json --strip`.

`--outline` 시그니처 정확도 (33쿼리 자체 벤치마크): **100% well-formed** (paren 균형, 잘림 없음).

## `digest` — 빌드 / 테스트 / CI 출력 압축

흔한 도구들의 출력을 자동 감지해서 압축합니다. **에러와 실패는 절대 손실 없음** — 진행 라인만 카운트로 축약됩니다.

**지원 핸들러:** `cargo`, `pnpm`/`npm`/`yarn`/`bun`, `tsc`, `pytest`, `go test`, `gradle`, `ruff`, `mypy`, `clang`/`gcc`/`cmake`/`make`/`swiftc` (`compiler`), GitHub Actions (`ci`). 알 수 없는 형식은 원본 그대로 통과.

15개 실제 fixture 측정:

| 도구 | raw → digest | 절감 |
|---|---|---|
| `cargo build` (clean, 218 crates) | 7,611 B → 59 B | **-99.2%** |
| `cargo test` (45 passing) | 3,368 B → 369 B | -89.0% |
| `pnpm install` | 1,323 B → 349 B | -73.6% |
| `tsc` (13 errors, 5 codes) | 1,085 B → 648 B | -40.3% |
| `pytest` (4 failures) | 2,762 B → 2,330 B | -15.6% |
| **GitHub Actions log (rust-lang/rust 실패 CI, 실측)** | **3.3 MB → 35 KB** | **-98.9%** ⭐ |
| `go test` (panic + stack 포함) | 1,034 B → 475 B | -54.1% |
| `gradle test` (2 failures) | 1,232 B → 522 B | -57.6% |
| `ruff` (9 violations, 3 codes) | 624 B → 597 B | -4.3% |
| `mypy` | 336 B → 237 B | -29.5% |
| `clang`/`cmake`/`swift` compilers | ~600 B (진행 라인만 제거) | -3 ~ -30% |
| **TOTAL (15 fixtures)** | **3.33 MB → 43 KB** | **-98.7%** |

```bash
# 자동 감지 실패 시 핸들러 강제 지정
semble_rs digest --format ci  ci_log.txt
semble_rs digest --format gradle gradle_test.log

# 어떤 핸들러가 선택됐는지 확인
semble_rs digest --show-format my_log.txt
```

**보존 보장**
- `file:line:col`, traceback, panic stack, 실패 step 본문은 항상 유지
- 반복되는 에러 코드는 그룹화 (예: `TS2322` 9건 → 상위 3건 + `+6 more`)
- CI `##[group]` 블록: 성공한 블록은 한 줄로 축약, 실패한 블록은 끝 80줄 그대로 보존

## 지원 언어 (검색)

| 언어 | 검색 | AST 청킹 | 의존성 그래프 |
| --- | --- | --- | --- |
| Rust | ✓ | ✓ | ✓ |
| Python | ✓ | ✓ | ✓ |
| JavaScript | ✓ | ✓ | ✓ |
| TypeScript | ✓ | ✓ | ✓ |
| Go | ✓ | ✓ | ✓ |
| Java | ✓ | ✓ | ✓ |
| C / C++ | ✓ | ✓ | ✓ |
| **Kotlin** (v0.3.0+) | ✓ | ✓ | ✓ |
| Ruby, PHP, Swift, 기타 | ✓ | 줄 기반 fallback | — |

**한글 검색 지원** — BM25 토크나이저가 유니코드(`\p{L}`)를 지원해 한글 주석, 문서, 변수명도 키워드 검색 가능. 원본 `semble`은 ASCII만 인식.

## 검색 품질

기본 임베딩 모델(`potion-code-16M`) — 50쿼리 ground-truth 자체 벤치마크:

| 메트릭 | 점수 |
|---|---|
| Recall@1 | 70% |
| Recall@5 | 96% |
| **Recall@10** | **100%** |
| MRR | 0.81 |
| 한글 Recall@5 | 60% |

벤치마크/평가 스크립트는 [`semble-train/`](./semble-train)에 있습니다 (Python).

### 실험용: 임베딩 모델 교체

`SEMBLE_MODEL_PATH`로 로컬 [model2vec](https://github.com/MinishLab/model2vec) 출력 디렉토리(`tokenizer.json` + `model.safetensors`)를 지정:

```bash
SEMBLE_MODEL_PATH=/path/to/my-distilled-model semble_rs search "쿼리" ./my-project --compact
```

`SFR-Embedding-Code-400M_R`를 teacher로 distill해봤지만 기본 모델보다 못했습니다 (R@10 96% vs 100%, 한글 R@5 40% vs 60%) — 영어 코드 vocab만 있어 한글 BM25 + semantic 시너지가 깨졌기 때문. teacher의 vocab이 실제 코퍼스를 cover해야 합니다.

## AI 에이전트 통합

### 글로벌 `CLAUDE.md` (Claude Code) / `AGENTS.md` (Codex)

`~/.claude/CLAUDE.md` 와 `~/.codex/AGENTS.md`에 다음과 같이 추가:

```markdown
# semble_rs — grep, cat, read, ls 대체 + 빌드/CI 출력 압축

코드 탐색 시 raw grep/cat/find/read 대신 이걸 사용:

  semble_rs search "<feature>" /path --outline      # 1단계 탐색
  semble_rs search "<symbol>"  /path --compact      # 2단계 정밀 탐색
  semble_rs deps   <file>      /path
  semble_rs impact <file>      /path

빌드 / 테스트 / CI 출력은 항상 `semble_rs digest`로 파이프:

  cargo build 2>&1   | semble_rs digest
  pnpm install 2>&1  | semble_rs digest
  pytest 2>&1        | semble_rs digest
  gradle test 2>&1   | semble_rs digest
  gh run view <id> --log-failed | semble_rs digest

규칙: 심볼명을 추측하지 말고 자연어로 기능 설명, 디렉토리 경로를 넘기기
(파일 경로 X), semble_rs 결과로 부족할 때만 `grep`으로 보충.
```

### 프로젝트 단위 (모든 에이전트)

프로젝트 루트의 `CLAUDE.md` / `AGENTS.md`에 같은 내용을 두면 Claude Code, Codex, Cursor (`.cursorrules`), Aider, OpenHands에서 모두 작동합니다.

## 라이선스

MIT — [LICENSE](./LICENSE) 참조.

크레딧:
- 원본: [MinishLab/semble](https://github.com/MinishLab/semble) — 이 Rust 포팅의 기반이 된 Python 구현
- 임베딩 모델: [`minishlab/potion-code-16M`](https://huggingface.co/minishlab/potion-code-16M)
- 의존 라이브러리: `tree-sitter`, `ndarray`, `safetensors`, `tokenizers`, `hf-hub`, `ignore`, `clap`
