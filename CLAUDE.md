# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a personal LaTeX CV / cover letter repository for Gabriele Canò Cantelmo, not a software application. There is no application code, build tooling, or test suite — the only "source code" is LaTeX, and the only deliverables are PDFs.

## Layout

- `TeX/CV/main.tex` — the single English CV source, built with `moderncv` (casual style, blue).
- `TeX/CoverLetters/*.tex` — one `.tex` file per cover letter, typically named for the target company (e.g. `coverLetter_bendingSpoons2026.tex`). Each is a standalone `moderncv` letter using the same name/contact block as the CV.
- `PDFs/CV/ENG/`, `PDFs/CV/ESP/`, `PDFs/CoverLetters/` — output directories for generated PDFs (populated by CI, kept in git with `.gitkeep` placeholders otherwise).
- Root-level PDFs (`CV Gabriele Cano - 2025.pdf`, `CV2026 - Gabriele Cano Cantelmo.pdf`, `Cover Letter - Gabriele Cano Cantelmo.pdf`) are older/manually-placed exports, distinct from the CI-generated ones under `PDFs/`.

## Editing conventions

- Keep the name/title/address/email/social block at the top of `main.tex` and each cover letter in sync — they're duplicated per file (no shared preamble/include).
- CV entries use `\cventry` for experience/education/certifications and `\cvitem`/`\cvitemwithcomment` for skills/summary/languages. Follow the existing itemize formatting (`[leftmargin=*, topsep=0pt, partopsep=0pt, parsep=0pt, itemsep=2pt]`) for bullet lists inside an entry.
- A new cover letter is a new file under `TeX/CoverLetters/`; the CI workflow keys off newly **added** files in that directory (see below), so don't overwrite an existing letter in place if you want it rebuilt as a new PDF.

## CI/build pipeline

Three workflows, split so the translation and PDF-build stages are independently runnable (via their own `workflow_dispatch`) as well as chained together by the orchestrator:

- **`blank.yml`** (orchestrator) — triggers on push to `main` touching `TeX/CV/main.tex`, `TeX/CoverLetters/**`, or any of the three workflow files (also `workflow_dispatch`, which forces a full CV rebuild). Its `changes` job diffs the push to decide whether the CV changed and whether exactly one cover letter was **added** (fails if more than one was added in the same push — output filename would collide). It then calls `translate-cv.yml` (only if the CV changed) and `build-pdfs.yml` as reusable workflows (`uses: ./.github/workflows/...`), downloads the `generated-pdfs` artifact the build produces, and commits it back to `main` with `[skip ci]`.
- **`translate-cv.yml`** — fetches a GitHub OIDC token and uses it via Anthropic Workload Identity Federation (no static API key) to call `claude-sonnet-5` and translate `TeX/CV/main.tex` to European Spanish, preserving all LaTeX commands/URLs/company/product names. Uploads the result as the `cv-es-source` artifact (7-day retention). Runnable standalone.
- **`build-pdfs.yml`** — installs TeX Live (`texlive-latex-base/recommended/extra`, `texlive-fonts-extra` for moderncv's FontAwesome icons, English/Spanish language packs) and compiles PDFs with `pdflatex` (twice, for cross-reference resolution) into `PDFs/CV/ENG/`, `PDFs/CV/ESP/`, `PDFs/CoverLetters/`, each renamed with a `<YYYYMMDD>` date stamp. Takes `build_english` / `build_spanish` / `cover_letter_source` inputs so it can be run standalone against just the English CV (no Spanish artifact needed) to test LaTeX/package issues without spending Claude credits — `build_spanish` defaults to `false` on manual dispatch for that reason. Uploads everything built as the `generated-pdfs` artifact.

The Anthropic WIF federation rule (Console-side, not in this repo) matches on `repository`/`repository_owner` claims from the GitHub OIDC token — not `event_name` (differs between `push` and `workflow_dispatch`) and not a specific workflow file, so it authorizes both trigger types and any workflow in this repo.

There is a fourth, unrelated workflow (`anthropic-wif-test.yml`) that exercises the raw GitHub OIDC → Anthropic WIF token exchange manually; it isn't part of the CV build.

## Building locally

No local build scripts exist. To compile a document locally, run `pdflatex` twice (for cross-reference resolution) against the `.tex` file, e.g.:

```
pdflatex -interaction=nonstopmode -halt-on-error -output-directory <outdir> TeX/CV/main.tex
pdflatex -interaction=nonstopmode -halt-on-error -output-directory <outdir> TeX/CV/main.tex
```

Requires a TeX distribution with the `moderncv` package (e.g. MiKTeX or TeX Live full/extra).
