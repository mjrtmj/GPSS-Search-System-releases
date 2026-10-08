# GPSS 檢索系統 — AI 操作指南（gpss-cli）

版本: v0.5.3

本文件是給 **AI 代理**（Claude、ChatGPT、Codex、Copilot…）讀的操作合約。
使用者只要把這份文件貼給 AI，AI 就能透過命令列操作本工具檢索 TIPO GPSS 全球專利資料庫。
人類使用者請看 `使用說明.md`。

---

## 1. 你（AI）在操作什麼

- `gpss-cli.exe` 與 GUI 版 `GPSS檢索系統.exe` 在同一資料夾，共用同一份 `config.json`
  （GPSS 驗證碼）、`quota.json`（配額計數）與 `output/`（匯出結果）。
- 安裝版預設位置：`%LOCALAPPDATA%\Programs\GPSS檢索系統\gpss-cli.exe`。
  安裝時若勾選「加入 PATH」，任何目錄直接打 `gpss-cli` 即可。
- 免安裝版：解壓資料夾內的 `gpss-cli.exe`。
- 驗證碼：由使用者先在 GUI 的「設定」填好，或設環境變數 `GPSS_USER_CODE`。
  **你不需要也不應該知道驗證碼**；所有輸出、log、錯誤訊息都已遮蔽它。

## 2. 一律加 `--json`

```
gpss-cli --json <子命令> [參數]
```

- stdout **只有一個 JSON 物件**（UTF-8）；進度訊息走 stderr，可忽略。
  這條合約涵蓋一切：用法錯誤（`kind: usage`）、`--version`（`{"ok":true,"cmd":"version","version":"…"}`）、
  `--help`（`cmd: help`，`usage` 是文字說明）、非預期例外（`kind: internal`）。
- 成功：`{"ok": true, "cmd": "...", ...}`；失敗：`{"ok": false, "cmd": "...", "kind": "...", "error": "...", "fields": {...}}`。
- exit code：
  - `0` 成功或正常空結果
  - `1` GPSS 查詢失敗（`kind` 為 `auth` / `ip_blocked` / `quota_exceeded` / `network` / `gpss`）或內部錯誤（`internal`）
  - `2` 本機就能判定、你可以自行修正的問題：`config`（沒驗證碼）、`usage`（參數不對）、`invalid_query`（檢索式不對）

## 3. 子命令

| 子命令 | 用途 | 配額成本（「檢索結果」軌） |
|---|---|---|
| `reference` | 印出所有代碼表（檢索欄位、資料庫、案別、類型、輸出欄位、規則、exit code） | 0（離線） |
| `count` | 只回符合筆數 | 每次固定 30 筆 |
| `search` | 檢索並回傳書目（預設 30 筆，`--qty` 最多 10,000） | 取回筆數（最少 30） |
| `export` | 批次撈全部結果建台帳（CSV+JSON），超過 10,000 筆自動切窗 | **30（先 count）＋全部筆數**；切窗時每個窗再各加一次 30 的 count |

**注意另一條配額軌**：條件**只有** `--app-no` 或 `--pub-no`（沒有其他檢索欄位）時，
API 走「單筆案號」軌，限制小得多：上班每小時 300／時段 3,000，下班每小時 1,000／時段 10,000。
所以不要用 `export --app-no` 批量逐案撈；一次查多案請用其他欄位（申請人、日期…）走檢索軌。

**建議流程：先 `reference` 一次記住代碼 → `count` 看筆數 → 筆數合理再 `export`。**
不要跳過 `count` 直接 `export` 大範圍條件。

### 檢索條件參數（各子命令通用）

| 參數 | 欄位 | 說明 |
|---|---|---|
| `--applicant` | AX | 申請人（`--first-applicant` 為第一申請人） |
| `--inventor` | IV | 發明人 |
| `--agent` | LX | 代理人 |
| `--title` | TI | 專利名稱 |
| `--abstract` | AB | 摘要 |
| `--title-abstract` | TI/AB | 名稱或摘要（複合欄位，常用） |
| `--claims` | CL | 請求項 |
| `--app-no` / `--pub-no` | AN / PN | 申請號 / 公開公告號 |
| `--ipc` / `--cpc` | IC / CS | 分類號 |
| `--examiner` | EX | 審查委員 |
| `--priority` | PR | 優先權號 |
| `--pub-date` / `--app-date` / `--priority-date` | ID / AD / DR | 日期，格式 `起:迄`（`20240101:20241231`）或年 `2024` |

- 欄位之間是 AND。欄位內可用 `and` / `or` / `not`：`--title "無線裝置 or 通訊裝置"`。
- **不支援** `*` 切截與近接運算。
- 至少要給一個檢索欄位。

### 範圍參數

| 參數 | 預設 | 說明 |
|---|---|---|
| `--db` | `TWA,TWB,TWD`（本國公開／公告／設計） | 資料庫代碼逗號分隔；完整清單看 `reference`。無效代碼會被拒絕（API 對無效值靜默忽略，工具端擋下） |
| `--ag` | `A,B` | 案別：A 公開案、B 公告案 |
| `--ty` | 全部 | 類型：I 發明、M 新型、D 設計 |
| `--fields` | search 精簡／export 完整 | 輸出欄位代碼（expFld）；加 `CL` 取請求項全文（韓國庫沒有請求項） |
| `--qty` | 30 | 僅 search：取回筆數 30～10,000 |
| `--window-field` | `ID` | 僅 export：超過 10,000 筆時用哪個日期切窗（ID 公開日／AD 申請日） |
| `--output DIR` | 程式旁 `output/` | 僅 export：輸出根目錄，每次匯出在其下建「時間戳_檢索式摘要」子資料夾 |

## 4. 輸出 JSON 形狀

`count`
```json
{"ok": true, "cmd": "count", "total": 1063, "total_display": "1,063", "fields": {"TI": "觸控筆"}}
```

`search`（鍵完整）
```json
{"ok": true, "cmd": "search", "fields": {"TI": "觸控筆"},
 "status": "success", "total": 1063, "returned": 30, "truncated": false, "message": null,
 "windows": [],
 "records": [{"database": "本國公告", "kind": "B", "patent_type": "發明",
              "publication_no": "TWI803321B", "publication_date": "20230521",
              "application_no": "TW111131346", "application_date": "20220822",
              "title": "…", "title_en": "…", "abstract": "…",
              "applicants": [{"name": "…", "english_name": "…", "country_code": "TW"}],
              "inventors": [{"name": "…", "english_name": null, "country_code": null}],
              "agents": ["…"], "examiners": ["…"],
              "ipc": ["…"], "cpc": ["…"], "loc": [], "uspc": [], "fi": [], "f_term": [], "d_term": [],
              "citations": [],
              "priority_claims": [{"country": "US", "doc_number": "…", "date": "20210101"}],
              "claims": [{"num": "1", "text": "…"}]}]}
```
每筆 record 的鍵固定是上面這 24 個；沒資料的是 `null` 或 `[]`（`loc`／`uspc`／`fi`／`f_term`／`d_term`
只有設計、美國、日本庫才有值）。`status` 為 `no_record` 時是**正常空結果**（`ok` 仍 true、`records` 為 `[]`）。
`truncated: true` 表示 total 大於實際取回（配額中斷或超過單式上限）。

`export`（鍵完整；**不含 records**——可能上萬筆，請讀 `files.json` 或 `files.csv`）
```json
{"ok": true, "cmd": "export", "fields": {"TI": "觸控筆"},
 "status": "success", "total": 12111, "returned": 12111, "truncated": false, "message": null,
 "windows": [{"range": "20230101:20230630", "total": 6000, "splittable": true, "fetched": 6000}],
 "output_dir": "C:\\...\\output\\20260908_1530_TI=觸控筆",
 "files": {"csv": "...\\result.csv", "json": "...\\result.json", "appnos": "...\\appnos.txt",
           "query_info": "...\\query_info.json", "search_log": "...\\search_log.jsonl"}}
```
`files` 只列實際存在的檔（沒有申請號時沒有 `appnos`）。`status` 為 `no_record` 時 `output_dir` 為 `null`、`files` 為 `{}`。

`reference`：`ok`、`cmd`、`version`、`search_fields`、`cli_args`（參數名 → 欄位代碼）、
`databases`（每庫 `name`、`has_claims`）、`ag`、`ty`、`exp_fld`、`defaults`、`rules`、`exit_codes`。

錯誤：`ok`、`cmd`、`kind`、`error`，以及（已載入設定後）`fields`；`usage` 類另有 `usage` 文字。

## 5. 錯誤 `kind` 與你該怎麼做

| kind | 意思 | 你該做的事 |
|---|---|---|
| `config` | 沒有驗證碼或 config.json 壞掉 | 停下來請使用者在 GUI「設定」填驗證碼。**不要**自己猜或生成驗證碼 |
| `auth` | 驗證碼不存在／過期 | **立刻停止，絕對不要重試**。連續嘗試無效碼會讓整台電腦的 IP 被 TIPO 封鎖。回報使用者 |
| `ip_blocked` | IP 已被封 | 停止；請使用者聯絡 gpss-service@tipo.gov.tw 解封 |
| `quota_exceeded` | 本時段配額用罄 | **不要重試硬打**。已取回的部分已保留在 output/。告訴使用者等配額回補（時段界線：週一～五 08:00／18:00）再續跑，大批匯出建議排平日 18:00 後或假日 |
| `invalid_query` | 檢索式問題（無效欄位、太長、禁用字元、切截符號…） | 修正條件後再試；`error` 有說明 |
| `network` | 連線失敗／TIPO 端 5xx | 可稍後重試一次；持續失敗就回報 |
| `gpss` | 其他 API 錯誤 | 回報 `error` 原文給使用者 |

### 錯誤訊息可能被遮蔽

所有錯誤訊息、`fields` 回聲、輸出路徑都經過憑證遮蔽：已知驗證碼、以及任何
**16 碼以上、字母數字兼具的連續英數字串**（含緊鄰其他英數的情況）整段變成 `***`
（防止誤貼的驗證碼外洩）。書目資料（`records`、result.json／csv）不受影響。
若錯誤訊息裡看到 `***`，那是遮蔽，不是資料損壞；回應會帶 `fields_redacted: true`，
`query_info.json` 會帶 `query_redacted: true`，表示原查詢被遮過。回應裡的本機路徑
（`output_dir`、`files`）也會遮，遮過時帶 `paths_redacted: true`——此時請用你自己
給的 `--output` 推算檔案位置，或直接列該目錄下最新的子資料夾。
`--output` 你打的路徑原文若含這種字串會被直接拒絕（`kind: invalid_query`），請換路徑；
合法的長識別字資料夾名（如 `Project20260908Release`）也會被誤判，這是憑證鐵律的代價。
程式所在目錄、目前工作目錄、使用者家目錄這三個「可信根」的祖先資料夾名不算，
其餘部分才檢查。`config.json` 或 `GPSS_OUTPUT_DIR` 給的輸出路徑在載入時就會檢查
（含碼 → `kind: config`）。

## 6. 配額紀律（TIPO 官方限制，你必須遵守）

以「筆數」計，**每次呼叫最少扣 30 筆**，兩條軌各自獨立計算：

| 時段 | 檢索結果軌（一般條件） | 單筆案號軌（條件只有 `--app-no`／`--pub-no`） |
|---|---|---|
| 上班（週一～五 08:00–18:00） | 時段共 10,000 筆 | 每小時 300／時段 3,000 |
| 下班（其餘時間與假日） | 時段共 30,000 筆 | 每小時 1,000／時段 10,000 |

- 時段界線（08:00／18:00）與整點回補；用罄時 `kind: quota_exceeded`，已取回部分保留。
- `export` 成本 = 30（count）＋ 全部筆數；超過 10,000 筆會切窗，每窗再各一次 30 的 count。
- 單一檢索式上限 10,000 筆。
- 因此：**先 count、再決定 export**；不要為了「看看」重複跑 search；
  同一條件不要重複 export，結果都在 `output/`，先找找看。
- 工具內建 0.5 秒最小呼叫間隔，你不需要自己 sleep。

## 7. 已知資料特性（不是 bug）

- 「No record found」是正常空結果。但若條件裡含 `count(`、`sys.`、`${`、`alert(`、`(select`、
  `sleep(`、撇號 `'` 等字串，API 會靜默回零結果；工具送出前會擋，命中時 `kind` 為 `invalid_query`。
- 韓國庫（KPA／KPB／KPD）沒有請求項（`reference` 的 `databases[].has_claims` 為準）。
- 美國庫只有英文名稱（`title` 可能為 null、`title_en` 有值）；判斷名稱請用 `title or title_en`。
- 日期一律 `YYYYMMDD` 字串。

## 8. 範例

```
gpss-cli --json reference
gpss-cli --json count --applicant 台積電 --pub-date 2024 --ag B
gpss-cli --json search --title-abstract 半導體 --pub-date 20240101:20240131 --db TWA,TWB --ty I --qty 100
gpss-cli --json export --applicant "台達電子" --pub-date 20230101:20241231 --fields PN,AN,ID,AD,TI,AB,PA,IN,IC,CS,PR,CL --output D:\patent-work
gpss-cli --json search --pub-no TWI803321B --fields PN,AN,ID,AD,TI,AB,PA,IN,IC,CL   （單案看書目＋請求項；走單筆案號軌）
gpss-cli --json --version
```

原始碼版同義：`python -m gpss_search.cli --json ...`（在專案根目錄執行）。
