# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A personal study-content repo, not a software project. It contains Korean-language lecture series (Markdown source, most with a rendered interactive HTML set) teaching CS fundamentals — DB, Network, OS, C++, GameMath — to one specific learner: an Unreal C++ client-programming new hire preparing for job-change interviews. There is no build, lint, or test tooling; the only "correctness" bar is whether new lecture files match the established structure, tone, and content rules.

## The authoritative spec: 강의작성_지침.md

**[강의작성_지침.md](강의작성_지침.md) is the full spec for writing or continuing any lecture and takes precedence over general instinct.** Read it before authoring content. Key points (see the file for full detail):

- **Audience (never forget):** one Unreal C++ client-dev new hire, CS knowledge mostly forgotten, studying to answer interview questions confidently. Write as if explaining 1:1 to this one person — never "readers"/"you all" phrasing.
- **Tone:** 반말체 (informal register) in body text; interview-answer text inside "면접에서 나오면" sections uses polite 존댓말, on purpose — that tone contrast is intentional, not an inconsistency to fix.
- **Code examples must be C++/Unreal only** (`FString`, `TArray`, `int32`, `FCriticalSection`, `FScopeLock`, etc.), except the DB series which uses SQL. Python/Java/JS/Go/Rust/C# examples are forbidden, even as comparisons beyond a one-line remark.
- **Standard lecture skeleton** (in order): 목표/선수지식 header quote → 도입 (game-dev scenario or broken code) → 개념 설명 → diagram/code walkthrough → 코드 실전 예제 → 흔한 오해 (3–5 misconceptions) → 면접에서 나오면 (5–7 Q&A) → 셀프 체크 (4–6 items) → 다음 강의 예고. Every concept must be tied to a concrete Unreal/game-dev scenario (see the 지침's 개념→게임 연결 table) — no bare abstract explanations.
- **File naming:** main lectures `NN_제목.md` (zero-padded two-digit number + Korean title, spaces as `_`); supplements `NN-N_제목.md` or `특별강의_제목.md`.
- **Length target:** 300–500 lines, ~15–25 min read. Split overlong lectures into a supplement file rather than growing one file indefinitely.
- **Workflow gates:** for a brand-new subject, create `<과목>/강의계획서.md` first and get explicit user approval before writing 1강. During an existing series, only write as many lectures as the user asked for in that turn, then stop for confirmation ("계속" = proceed to next). Don't create new files unprompted — extend an existing lecture as a supplement if the gap can be filled that way.
- **Do not modify 강의작성_지침.md itself** without an explicit user request; if an existing lecture's tone conflicts with the 지침, the existing lecture wins (series consistency over spec literalism).

## Repository structure

```
DB/lectures/NN_제목.md            ← DB series (SQL/PostgreSQL), 20 lectures. No 강의계획서.md, no HTML.
Network/강의계획서.md              ← syllabus, read first for scope/ordering
Network/lectures/NN_제목.md       ← Network series, 15 lectures + 보너스 (Markdown source)
Network/lectures_html/            ← HTML render: 15 lectures + index.html (보너스 has no HTML yet)
OS/강의계획서.md
OS/lectures_md/NN_제목.md          ← OS series, 15 lectures + 보너스 (Markdown source)
OS/lectures_html/NN강_제목.html    ← HTML render of every OS lecture + 보너스 + index.html
Cpp/강의계획서.md, Cpp/lectures/, Cpp/lectures_html/          ← Cpp series, 12 lectures + 보너스
GameMath/강의계획서.md, GameMath/lectures/, GameMath/lectures_html/  ← GameMath series, 8 lectures + 보너스
AI면접_대비_학습순서.md            ← tiered (S/A/B/C) study order across all series; the HTML conversion followed it
```

Series with HTML use two folders: the Markdown source (`lectures/` or `lectures_md/` for OS) and `lectures_html/` (rendered, `NN강_제목.html`). The Markdown is the source of truth; the HTML is a hand-ported, standalone dark-theme page for studying. Cpp/Network/GameMath HTML covers every numbered lecture; the `보너스_*_BEST20` files are Markdown-only except OS. Each HTML file is a self-contained single-page document (inline `<style>`/`<script>`, Pretendard webfont via CDN, dark theme, fixed nav + scroll-spy, misconception accordions, flip Q&A cards, localStorage self-check) and carries one bespoke interactive "killer widget" for its topic. There is no build script in the repo; if asked to add/update an HTML version, copy the structure and CSS variables of a sibling file in the same `lectures_html/` folder (each Network Part / Cpp / GameMath set has its own `--acc` accent color) rather than introducing a new template or toolchain, and keep that folder's `index.html` entry and the prev/next footer links of adjacent lectures in sync.

`강의계획서.md` (per subject) is the curriculum table — lecture numbers, titles, and the keywords each lecture must cover. Cross-check new lecture content against its row before considering the lecture done.

Two paths referenced by `강의작성_지침.md` (`DB/init.sql`, `DB/lectures/특별강의_init_sql_분석.md`) are listed in `.gitignore` and are not present in the working tree — they are optional local artifacts, not missing files to recreate unprompted.

## Common tasks

- **Continue an existing series:** read the subject's `강의계획서.md` for the next lecture's keywords, check the immediately preceding lecture file for tone/format continuity, then write following the skeleton above.
- **Start a new subject:** confirm scope with the user, write `<과목>/강의계획서.md`, get approval, then begin 1강.
- **Add/adjust an HTML render:** base it on a sibling `<과목>/lectures_html/*.html` file (same accent colors, nav, self-check, footer pattern), add one bespoke widget for the topic, and keep `index.html` and adjacent prev/next links in sync.
