---
name: russian-zh-visible-pdf-layout-qa
agent_created: true
description: QA and rebuild Russian-to-Simplified-Chinese visible PDFs when text overflows, wraps incorrectly, or drops content. Works with the naval-history DJVU pipeline in D:\ocr_work\translate.
---

# Russian-to-Chinese Visible PDF Layout QA / Rebuild

Use this skill when a Russian-source visible-Chinese PDF (e.g., `_zh.pdf` in the naval-history pipeline) shows:
- characters overflowing the right or bottom page edge,
- dropped / missing text,
- encoding errors or misplaced line breaks,
- unreadably tiny font from a global wrap-width bug.

## Workspace and key files

The pipeline lives in `D:\ocr_work\translate`.

- `build_zh_visible.py` — final builder. Reads `D:\Desktop\<hash>_searchable.pdf` + `zh\<hash>_zh_full.txt`, burns visible Chinese over the original page.
- `layout_common.py` — shared geometry engine: text-block detection, per-block wrapping, fallback full-page reflow, base font sizing.
- `qa_v2.py` — single-book validator. Replays geometry and reports OVERFLOW_X / OVERFLOW_Y / LEFTOVER / CRAMPED / ENCODING issues.
- `launch_qa.py` / `qa_controller.py` — detached QA controller, one short subprocess per book.
- `launch_rebuild.py` / `rebuild_controller.py` — detached rebuild controller, rebuilds all books sequentially.
- `render_check.py` — render selected pages to PNG for visual spot-check.
- `qa_v2_report.txt` / `rebuild_log.txt` — progress logs.

## Geometry rules

- Wrap width is per detected text block, never global. This fixes the original overflow bug where a narrow right column was filled using the widest-column width.
- Left indent is unified to `PAD = 4.0` pt; text is left-aligned to `block.x0 + PAD`.
- Minimum readable font floor `MIN_SIZE = 5.6` pt. Pages that hit the floor are flagged CRAMPED but keep all text.
- If text still does not fit inside detected blocks, the engine falls back to full-page reflow (`_flow_full_page`) so no text is dropped.
- Encoding is forced UTF-8 everywhere; markers use `【第N页】` where N is 0-based page index.

## Workflow

1. Check current state: open `qa_v2_report.txt` if it exists, or run detached QA:
   `python launch_qa.py`
2. Inspect the report. Goal: zero OVERFLOW_X, OVERFLOW_Y, LEFTOVER, ENCODING. CRAMPED is acceptable if the font remains legible.
3. If issues remain, edit `layout_common.py` (PAD / MIN_SIZE / line-height / fallback logic), then re-run QA.
4. When QA is clean, rebuild all books:
   `python launch_rebuild.py`
5. Watch `rebuild_log.txt`. Every book should end with `rc=0`.
6. Verify: render representative pages with `python render_check.py <hash> <pno1,pno2,...>` and visually confirm no overflow.

## Detached subprocess pattern

Long builds hit the agent background-task timeout. Always launch the QA or rebuild controller with `launch_*.py`, which uses `subprocess.Popen(..., creationflags=0x00000008)` (DETACHED_PROCESS). The controller then spawns one short `build_zh_visible.py` subprocess per book, so no single call exceeds the timeout.

## Inputs / outputs

- Inputs: `D:\Desktop\<hash>_searchable.pdf`, `D:\ocr_work\translate\zh\<hash>_zh_full.txt`.
- Output: `D:\Desktop\<hash>_zh.pdf` (overwritten in place).
- Final verification PNGs land in `D:\ocr_work\translate\chk\`.
