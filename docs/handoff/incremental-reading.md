# 交接：漸進閱讀（Incremental Reading）伴生引擎

分支：`feat/incremental-reading`（從 `Dev` 切出；`Dev` 是默認分支，PR 合回 `Dev`）。

## 目標

在 Engram 裏加一條**輸入側**流水線，與現有的**輸出側**（`/learn` `/review` `/coach`，閉卷回憶 + FSRS）銜接成一個閉環：

```
原始材料 → AI 教學化改編 → 漸進閱讀 + 切片(extract) → FSRS 排程切片與摘要
        → AI 把成簇切片合成(synthesis)新文章 → 新文章回到閱讀隊列（遞歸）
        → 消化完成的切片沉澱(distill)為 engram 節點，進入 /review
```

用戶要的五個能力，每個都必須有落點：FSRS 排程、AI 生成教學文章、切片、AI 組合切片成新文章、FSRS 驅動的漸進摘要。

## 架構決定（已定，不要重開）

- **伴生（companion），不魔改。** 新增 `scripts/reader.py`，不修改 `scripts/engram.py`（1.3 萬行，自帶 selftest、收據和盲評體系，它完全沒有文章或切片的概念）。這樣才能繼續 `git merge upstream/main`（upstream = `https://github.com/nagisanzenin/engram`）。
- **確定性邊界照搬 engram 憲法第 10 條。** 日期、stability、difficulty、due 只由 `reader.py` 計算；LLM 只負責寫文字（改編、摘要、合成），再通過 CLI 寫回。
- **沒有網絡。** `reader.py` 只用 stdlib，零網絡模塊。`engram.py` 的 selftest 只掃描它自己（`scripts/engram.py:9717-9760`，用 AST 檢查 `__file__`），所以 `reader.py` 要給自己寫一份同樣的 AST 禁令檢查。調用 AI 的永遠是 agent，不是腳本。
- **FSRS 復用，不復制。** 從 `engram.py` import 純函數：`apply_rating(fsrs, rating_name, on_date, cap_days=None, w=None)`（`scripts/engram.py`「fsrs core」段）、`interval_for`、`retrievability`、`today()`（尊重 `ENGRAM_TODAY`）、`home()`（尊重 `ENGRAM_HOME`）。開工前先確認 import `engram` 沒有副作用。
- **數據放在 engram home 下的獨立子目錄**（建議 `home()/reading/`），不碰 `graphs/` 和 receipts。

## 數據模型（起點，可以改，改了要寫明理由）

- `source`：導入的原始材料或 AI 改編後的教學文章。字段：`id, title, origin(url|path|synthesized), parent_slices[], priority(0-100), cursor`，正文存成 Markdown。
- `slice`：從 source 切出的片段。字段：`id, source_id, anchor(在原文中的位置), text, tags[], stage(raw→digest→axiom), summary_history[], fsrs{s,d,last,due,reps,lapses}`。
- **漸進摘要（progressive summarization）**：每次複習切片時，agent 寫一版更短的摘要，`stage` 往前推進；`axiom` 階段的切片才有資格 `distill`。
- **合成觸發**：按 tag 或主題聚類，同簇內已消化的切片數 ≥ 閾值（默認 5）就可以合成；合成出的文章作爲新的 `source` 入隊，`parent_slices` 記錄它的來源。

## 步驟

每一步做到完成條件纔算完。

1. **摸底。** 讀 `scripts/engram.py` 的 fsrs core 段、`home()`/`p()`/`load_payload()`、`cmd_add_topic`（payload 必須有 `topic,title,nodes,order`；`--extend` 只能追加），再讀 `pi/engram.ts` 和 `skills/review/SKILL.md`。
   完成：寫出 distill 要提交給 `add-topic --extend` 的節點 JSON 最小樣例，並用 `ENGRAM_HOME=$(mktemp -d)` 實跑通過。
2. **`scripts/reader.py` CLI。** 子命令至少有 `add-source`、`next`（今天到期的 source 段落和切片，按 priority 加 due 排序）、`extract`、`rate`（複習切片，同時寫入新摘要並推進 stage）、`clusters`（列出可合成的簇）、`add-synthesis`、`distill`（生成 engram 節點 payload）、`selftest`。
   完成：`python3 scripts/reader.py selftest` 全綠，覆蓋排程、stage 推進、聚類閾值、無網絡 AST 檢查、`ENGRAM_TODAY` 凍結日期。
3. **Skills。** 新增 `skills/read/SKILL.md`（伴讀：教學化改編 → 分段推送 → 切片 → 漸進摘要）和 `skills/synthesize/SKILL.md`（取簇 → 合成 → 消解矛盾 → 入隊）。對應 prompt 放在 `pi/prompts/read.md`、`pi/prompts/synthesize.md`，格式照抄 `pi/prompts/review.md`。寫法參照 `skills/_shared/dialogue-grammar.md`：agent 是伴讀，用戶親自判斷哪些內容值得切。
   完成：在 pi 裏用一篇真實文章走完 read → extract → rate → synthesize → distill → `/review` 能看到新節點。
4. **提醒。** 擴展 `pi/engram.ts` 的 session-start nudge，讓它同時報告到期的閱讀數量，例如 `[engram] 3 reading due · 5 reviews due`。整份 contract 不變：每個 session 最多提醒一次，任何失敗都降級爲靜默。
   完成：`npx vitest run __tests__/session-start.test.ts` 綠，並爲新文案補一條測試。
5. **收尾。** README 加一節說明閱讀流水線；`python3 scripts/engram.py selftest` 和 `npx vitest run` 全綠。

## 注意事項

- **工作區噪音**：`/mnt/a/...` 是 Windows 檔案系統，`git status` 會把大約 127 個文件顯示爲已修改，實際只是 CRLF 和權限差異（`git diff --ignore-all-space` 爲空）。只 `git add` 你自己改的文件，不要 `git add -A`。
- 遞歸合成會產生重複：給合成文章加去重規則（同簇再次合成時，必須有新切片加入）。
- 防止切片債務（extract bankruptcy）：`next` 每天推送有上限（默認 source 段落 5、切片 10），超出的內容順延。
