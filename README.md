# Taiwan Medical Exam Skills

> 給 Claude 用的台灣醫師國考相關 skill 集合 — Claude skills for Taiwan medical board exam workflows.

## 這個 repo 是什麼？

收錄一系列 [Claude Skill](https://www.anthropic.com/news/claude-skills)，把台灣醫師國考（醫學三、四、五、六）相關的繁瑣處理流程包好，讓 Claude 直接接手。

目前收錄：

| Skill | 功能 | 路徑 |
|------|------|------|
| `anki-flashcard-medical-exam` | 把考古題（PDF / RTF / MD / TXT / DOCX）轉成 Obsidian-to-Anki flashcard 格式，自動加 `[[wikilink]]` 與科別 tag | [`skills/anki-flashcard-medical-exam/`](skills/anki-flashcard-medical-exam/) |

之後會持續加入其他相關 skill。

## 怎麼安裝？

### 在 Claude Code / Cowork 使用

1. clone 這個 repo 到本機：
   ```bash
   git clone https://github.com/g07101334-max/medexam-anki-skill.git
   ```
2. 把想用的 skill 資料夾複製到你的 Claude skills 目錄：
   - macOS：`~/Library/Application Support/Claude/skills/`
   - 或是 Claude Code plugin 結構底下的 `skills/` 目錄
3. 重新啟動 Claude Code / Cowork session，skill 就會被讀到。

### 直接看內容

每個 skill 的 `SKILL.md` 就是完整文件，包含什麼時候會被觸發、操作流程、格式規則。可以直接讀懂，當作自己手動操作時的 SOP。

## 怎麼用 `anki-flashcard-medical-exam`？

詳細請看 [`skills/anki-flashcard-medical-exam/SKILL.md`](skills/anki-flashcard-medical-exam/SKILL.md)。摘要：

1. 跟 Claude 說「幫我把這份考古題做成 flashcard」，並提供：
   - **輸入檔案路徑**（PDF / RTF / MD / TXT / DOCX 都可以）
   - **Obsidian vault 中要存放 flashcard 的資料夾路徑**
2. Claude 會自動：
   - 辨識考卷代號（如 `115-1-4`）
   - 詢問要一次生 80 題還是分批
   - 依副檔名選對應 parser（pdfplumber / striprtf / python-docx / 直接 Read）
   - 產出符合 [Obsidian-to-Anki](https://github.com/Pseudonium/Obsidian_to_Anki) 套件格式的 `.md`
   - 自動標註疾病、症狀、藥物、病原體、檢驗異常等 `[[wikilink]]`
   - 加上科別 tag（如 `[[CV考古]]`、`[[Anes考古]]`）
3. 寫入 `{你指定的資料夾}/醫學X/<代號>/<代號>.md`

之後在 Obsidian 開啟 Obsidian-to-Anki sync，就能匯入 Anki。

## 依賴

- [Obsidian](https://obsidian.md/)
- [Obsidian-to-Anki](https://github.com/Pseudonium/Obsidian_to_Anki) 套件
- Claude Code 或 Cowork
- Python 套件（依輸入格式自動安裝）：`pdfplumber` / `striprtf` / `python-docx`

## 想貢獻？

歡迎 PR。如果你也在準備國考、有自己的 tag 規則、或想新增別科 skill（如分科整理、特定章節 cloze 標記等），開 issue 討論或直接送 PR。

## License

[MIT](LICENSE) © 2026 Ting Chiu

---

# English Summary

A collection of [Claude Skills](https://www.anthropic.com/news/claude-skills) for handling Taiwan's national medical licensing exam workflows.

## Skills

| Skill | Purpose |
|------|------|
| `anki-flashcard-medical-exam` | Converts past exam papers (PDF / RTF / MD / TXT / DOCX) into Obsidian-to-Anki flashcard format, auto-tagging diseases, symptoms, drugs, pathogens, and clinical specialties with `[[wikilinks]]`. |

## Installation

```bash
git clone https://github.com/g07101334-max/medexam-anki-skill.git
```

Then copy the desired skill folder into your Claude skills directory (e.g. `~/Library/Application Support/Claude/skills/` on macOS, or under your Claude Code plugin's `skills/` directory).

## Usage

Each `SKILL.md` is fully self-contained — it documents trigger conditions, workflow steps, formatting rules, and tagging conventions. To use `anki-flashcard-medical-exam`, simply ask Claude to convert an exam paper into flashcards and provide:

1. The exam paper's file path (PDF / RTF / MD / TXT / DOCX)
2. The Obsidian vault folder where the output should land

Claude handles file parsing, exam ID recognition, batching, tag scanning, and writing the final `.md` file into `{your_folder}/醫學X/<exam-id>/<exam-id>.md`.

## License

[MIT](LICENSE) © 2026 Ting Chiu
