---
name: anki-flashcard-medical-exam
description: 把台灣醫師國考（醫學三、四、五、六）的題目和答案轉換成 Obsidian to Anki flashcard 格式的 .md 檔案。支援多種輸入格式（PDF、RTF、Markdown、TXT、DOCX），輸出位置由使用者指定 Obsidian vault 資料夾。當使用者上傳/指定考古題檔案並要求製作國考 flashcard、Anki 卡片、或 Obsidian 筆記時，必須使用此 skill。也適用於使用者說「幫我做成 flashcard」、「繼續生題目」、「修正 tag」等情境。若使用者尚未提供檔案或輸出位置，務必先詢問再開始。
---

# 國考醫學 Flashcard 製作 Skill

## 任務
將台灣醫師國考醫學三（試題代號 1302）、醫學四（試題代號 2302）、醫學五、醫學六的題目和答案轉換成 Obsidian to Anki 的 flashcard 格式。

---

## 步驟

### Step 0：確認輸入檔案與輸出位置（必做，沒給就要問）

開始任何工作前，必須先確認兩件事。**若使用者已在訊息中明確提供，可跳過詢問；否則一定要先問**。

#### 0-1. 輸入檔案路徑

接受的檔案格式（透過副檔名自動辨識，不問使用者要哪種解析方式）：

| 副檔名 | 來源例 | 解析方式 |
|------|------|------|
| `.pdf` | 考選部官網原始 PDF（推薦，最準確） | pdfplumber 或 PyMuPDF |
| `.rtf` | 自行轉檔的 RTF | striprtf |
| `.md` / `.markdown` | 已轉純文字 | 直接 Read |
| `.txt` | 純文字稿 | 直接 Read |
| `.docx` | Word 檔 | python-docx |

**詢問範本**（使用者沒給檔案時）：
> 「請提供考古題檔案的完整路徑（支援 PDF / RTF / Markdown / TXT / DOCX）。如果題目和答案是分開的檔案，請一併提供答案卷路徑。」

#### 0-2. 輸出資料夾路徑（Obsidian vault 中存放 flashcard 的位置）

**詢問範本**（使用者沒給輸出位置時）：
> 「請提供 Obsidian vault 中要存放這份 flashcard 的資料夾絕對路徑（例如：`/Users/your_name/MyVault/考古/歷屆考古`）。我會在底下依「醫學X/代號/代號.md」自動建立子資料夾。」

確認後，本份 skill 會把最終 .md 寫到：

```
{使用者指定的 vault 資料夾}/醫學{三/四/五/六}/<代號>/<代號>.md
```

#### 0-3. 確認後再進入下一步
兩個路徑都拿到後，先 echo 給使用者確認：
> 「我會從 `{輸入路徑}` 讀取，產出後寫到 `{輸出資料夾}/醫學X/<代號>/<代號>.md`。沒問題的話我繼續。」

---

### Step 1：確認考試代號（必做）

從輸入檔案內標題或檔案名稱中，辨識本份考卷對應的代號。

**規則**：`年份-次數-X`
- 年份 = 民國年（114、115...）
- 次數 = 第一次 / 第二次（1 或 2）
- X = 科目編號（3 = 醫學三、4 = 醫學四、5 = 醫學五、6 = 醫學六）

範例：
- 「114 年第二次」醫師(二)醫學(三)（試題代號 1302） → `114-2-3`
- 「115 年第一次」醫師(二)醫學(四)（試題代號 2302） → `115-1-4`

統一加上 `[[]]` 變成 backlink，如 `[[114-2-3]]`、`[[115-1-4]]`。

**確認後告知使用者**：「這份是 115-1-4，正確嗎？」確認無誤再繼續。

---

### Step 2：詢問分批需求（必做）

**在開始生成之前，必須先問**：

> 「請問要一次生成全部 80 題，還是分批生成（例如每次 10 題或 20 題）？」

等待使用者回答後再繼續。

對於熟悉流程的使用者，可以一次問清楚分批方式 + 上次同樣的設定是否沿用（不寫 TARGET DECK、放到對應科目資料夾、使用既有科別 tag）。

---

### Step 3：讀取檔案（依副檔名分流）

讀取大型考古題檔案直接用 Read 工具可能會超過 token 上限，務必先用 bash + 對應的 parser 轉成純文字 .txt，再用 Read 讀取。

#### 3-A. PDF 檔案（推薦，國考官方格式）

```bash
pip install pdfplumber --break-system-packages 2>&1 | tail -1
python3 << 'PY'
import pdfplumber
src = "<輸入 PDF 絕對路徑>"
dst = "<暫存 .txt 絕對路徑>"
with pdfplumber.open(src) as pdf, open(dst, "w", encoding="utf-8") as out:
    for page in pdf.pages:
        out.write(page.extract_text() or "")
        out.write("\n\n")
print("done:", dst)
PY
```

若 pdfplumber 抓出來的文字排版亂掉（題號黏在一起、選項斷行錯亂），改用 PyMuPDF：

```bash
pip install pymupdf --break-system-packages 2>&1 | tail -1
python3 -c "
import fitz
doc = fitz.open('<輸入 PDF 絕對路徑>')
with open('<暫存 .txt 絕對路徑>', 'w', encoding='utf-8') as f:
    for p in doc:
        f.write(p.get_text())
        f.write('\n\n')
"
```

#### 3-B. RTF 檔案

```bash
pip install striprtf --break-system-packages 2>&1 | tail -1
python3 -c "
from striprtf.striprtf import rtf_to_text
with open('<輸入 RTF 絕對路徑>', 'r', encoding='utf-8', errors='ignore') as f:
    rtf = f.read()
text = rtf_to_text(rtf)
with open('<暫存 .txt 絕對路徑>', 'w', encoding='utf-8') as f:
    f.write(text)
"
```

#### 3-C. DOCX 檔案

```bash
pip install python-docx --break-system-packages 2>&1 | tail -1
python3 -c "
from docx import Document
doc = Document('<輸入 DOCX 絕對路徑>')
text = '\n'.join(p.text for p in doc.paragraphs)
with open('<暫存 .txt 絕對路徑>', 'w', encoding='utf-8') as f:
    f.write(text)
"
```

#### 3-D. Markdown / TXT 檔案

直接用 Read 工具讀取即可，不需要任何轉換。如果檔案很大可以分段讀。

---

#### 答案來源

- **答案內含於同一份檔案**：較新的醫學三/四考卷（如 114-2-3、115-1-4 之後）通常題目和 80 題答案在同一份檔案內
- **答案另一份檔案**：較舊的考卷或部分題型的答案需另外的 PDF/RTF 答案卷
- **答案排列**：題序 01–20、21–40、41–60、61–80 共四排

若一開始使用者只給了題目沒給答案，要主動詢問是否有獨立的答案卷。

---

### Step 4：解析答案

答案卷格式對應：

- `#` 符號代表該題有更正答案或特殊給分
- 備註欄說明特殊題目，例如：
  - `第X題答A或B或AB者均給分` → 寫成 `A 或 B（均給分）`
  - `第X題一律給分` → 寫成 `一律給分`
  - `第X題除未作答者不給分外，其餘均給分` → 依此說明

---

### Step 5：生成 Flashcard（按分批需求輸出）

#### 每題結構

```
[[年份-次數-X]]Q題號
題目文字
(A) 選項A
(B) 選項B
(C) 選項C
(D) 選項D #flashcard 
答案（解釋說明）[[tag1]], [[tag2]], [[科別考古]]
```

#### 重要格式細節

1. **題號獨立一行**：`[[115-1-4]]Q1` 單獨一行，題目文字從下一行開始
2. **題號格式**：`[[年份-次數-X]]Q題號`（X = 3、4、5、6 之科目編號）
3. 每個選項**各自獨立一行**
4. `(D) 最後選項 #flashcard ` → `#flashcard` 後面**空一個半形空格**再換行
5. 答案和所有 `[[]]` 標籤在**同一行**
6. `[[]]` 標籤之間用**逗號加空格**（`, `）隔開
7. 題目原文**完全照考卷不動**，不要在題目內加任何 `[[]]`
8. 題與題之間用 `---` 分隔
9. **檔案開頭不要寫 `TARGET DECK:` 行**。使用者已透過 Obsidian to Anki 套件的 folder/tag mapping 設定 deck，寫 TARGET DECK 反而會覆寫成單一 deck。直接以空行 + 第一題開始即可。

---

### Step 6：檔案存放位置（重要）

最終 .md 檔案放在 **Step 0-2 使用者指定的 vault 資料夾** 下，依科目分子資料夾：

```
{使用者指定的 vault 資料夾}/醫學三/<代號>/<代號>.md   ← 醫學三
{使用者指定的 vault 資料夾}/醫學四/<代號>/<代號>.md   ← 醫學四
{使用者指定的 vault 資料夾}/醫學五/<代號>/<代號>.md   ← 醫學五
{使用者指定的 vault 資料夾}/醫學六/<代號>/<代號>.md   ← 醫學六
```

舉例（假設使用者給的是 `/Users/alice/MyVault/考古/歷屆考古`）：

- 115-1-4 → `/Users/alice/MyVault/考古/歷屆考古/醫學四/115-1-4/115-1-4.md`
- 114-2-3 → `/Users/alice/MyVault/考古/歷屆考古/醫學三/114-2-3/114-2-3.md`

每個代號要建立**獨立子資料夾**（與檔名同名），檔案才能正確被 Obsidian to Anki 對應到對應的 Anki deck。

寫檔前先 `mkdir -p` 確保子資料夾存在。

---

### Step 7：Anki 同步注意事項（重要）

生成完 .md 後，使用者要在 Obsidian-to-Anki plugin 同步。**整個 vault scan 容易卡住（catastrophic backtracking / plugin hang on "Scanning vault"）**，因此務必提醒：

> 「同步前請先到 **Obsidian → Settings → Obsidian to Anki → Scan Directory**，把路徑改成你的科目資料夾（例如 `{你的 vault 資料夾}/醫學四`，路徑用相對於 vault 根目錄的相對路徑），不要留空白讓它掃整個 vault。」

要點：
- ✅ **設成科目資料夾**（如 `醫學四`）即可——所有年份子資料夾都會被掃到，新增的考古題自動納入
- ❌ **不要清空 Scan Directory**（清空 = 掃整個 vault，會卡在 Scanning vault 階段）
- ❌ **不要一次塞太多新檔再 sync**（可能觸發 plugin hang，分批同步較安全）

如果 Scan Directory 已設為某科目資料夾，新增的考古題 .md 會在下次 sync 時自動匯入到對應 deck，不需額外設定。

排錯：若同步後檔案內**完全沒長出 `<!--ID: ...-->` 註解**，代表 plugin 在 scanning 階段卡住，需照上述方式縮小 Scan Directory 範圍。

---

## [[]] 標籤規則

### 核心原則

**只標題目主文和選項 (A)(B)(C)(D) 裡實際出現的文字。**
**答案後的解釋說明（括號內說明原因的文字）不產生任何 [[]] 標籤。**
**⚠️ 中文疾病名和英文疾病名都要掃，不能只掃英文。**

標記對象分為以下六類：

#### 1. 疾病名稱（diseases）
- 單一疾病：如 `[[myocardial infarction]]`、`[[gout]]`
- **疾病大類/分類名稱**：若題目明確點名某疾病類別，也要標。例如題目說「原發性免疫缺陷疾病（primary immunodeficiency diseases）」→ 標 `[[primary immunodeficiency diseases]]`
- 所有選項裡的疾病都要標，**不論是否為正確答案**

#### 2. 症狀與臨床事件（symptoms & clinical events）
- 症狀：如 `[[fever]]`、`[[dyspnea]]`、`[[dark urine]]`
- **臨床事件**：如 `[[sudden death]]`（猝死）、`[[syncope]]`（昏厥）也要標
- 數值可推論症狀名：HR 130 bpm → `[[tachycardia]]`；RR 28 → `[[tachypnea]]`

#### 3. 體徵與理學檢查發現（signs & physical findings）
- 明確命名的體徵：如 `[[clubbing]]`、`[[systolic murmur]]`、`[[fine crackles]]`
- **選項中描述的聽診/叩診/視診發現也要標**：
  - 「聽診出現 bronchial sound」→ 標 `[[bronchial sound]]`
  - 「心音的 P2 消失」→ 標 `[[P2 disappearance]]`
  - 「奇脈（pulsus paradoxus）」→ 標 `[[pulsus paradoxus]]`
  - 「頸靜脈怒張（jugular vein engorgement）」→ 標 `[[jugular vein distension]]`
- **括號中英對照，取英文名稱標記**：如「皮膚膨脹程度偏低（dry skin turgor）」→ 標 `[[dry skin turgor]]`

#### 4. 藥物（drugs used for treatment）
- **每個選項中出現的每一種藥物都要標**，同一選項有多種藥物時全部都標
- 例如選項 A 寫「利尿劑、angiotensin-converting enzyme inhibitor、angiotensin receptor blocker 或 β-blocker」→ 全部都標：`[[diuretics]]`、`[[ACEi]]`、`[[ARB]]`、`[[beta-blocker]]`
- 排除：診斷性檢查專用藥（如壓力試驗用的 adenosine、dobutamine）

#### 5. 病原體（pathogens）
- **細菌、病毒、寄生蟲等病原體名稱出現在題目或選項中，必須標記**
- 例如選項列出 Shigella、Yersinia、Clostridium、Campylobacter → 全部標：`[[Shigella]]`、`[[Yersinia]]`、`[[Clostridium]]`、`[[Campylobacter]]`
- 不需另外標這些病原體造成的疾病（除非疾病名稱也有出現）

#### 6. 檢驗異常狀態（lab findings）
- 標**異常狀態名稱**：如 `[[hyperkalemia]]`、`[[hypocalcemia]]`、`[[metabolic acidosis]]`
- **不標**：單純的檢驗項目名稱（如 uric acid、creatinine 作為「要抽的項目」）
- **標**：明確異常狀態（如 hyperuricemia、azotemia）

---

### 不標記的項目
- ❌ **答案後括號內的解釋說明**（只有題目主文和選項才產生 tag）
- ❌ 診斷性檢查用藥（adenosine、dobutamine 等壓力試驗藥）
- ❌ 手術/處置名稱（appendectomy、colonoscopy 等）
- ❌ 影像檢查（X-ray、CT、MRI 等）
- ❌ 從知識推論但未出現在題目/選項文字中的內容
- ❌ 「cancer」等過於泛化的詞（標具體疾病名）

---

### 執行掃描的 Checklist（每題都要做）

生成每題 tag 前，依序執行：

**□ 題目主文掃描**
- [ ] 掃描題目的每一句話，包括括號內的英文對照
- [ ] 標出所有疾病名、症狀、臨床事件、體徵、藥物、病原體、疾病大類

**□ 選項逐一掃描（每個選項都要完整掃完）**
- [ ] (A) 選項：掃描每個字
- [ ] (B) 選項：掃描每個字
- [ ] (C) 選項：掃描每個字
- [ ] (D) 選項：掃描每個字

**□ 選項掃描常見漏標檢查**
- [ ] 每個選項的**所有藥物**都已納入（同一選項多種藥物全部標）
- [ ] 每個選項的**所有病原體**都已納入
- [ ] 括號內英文名已標記
- [ ] **體徵描述**（如「脈搏消失」「P2 消失」「bronchial sound」）已標記，不能因為是描述句就跳過
- [ ] **疾病修飾語不能丟棄**：「風濕性二尖瓣狹窄」→ `[[rheumatic mitral stenosis]]`，不能只標 `[[mitral stenosis]]`；「糖尿病腎病變」→ `[[diabetic nephropathy]]`，不能只標 `[[nephropathy]]`

---

### 科別標籤對照

**內科系（醫學三、四、五常用）**
| 科別 | 標籤 |
|------|------|
| 心臟科 | `[[CV考古]]` |
| 腸胃科 | `[[GI考古]]` |
| 腎臟科 | `[[Nephro考古]]` |
| 內分泌/代謝 | `[[Meta考古]]` |
| 胸腔科 | `[[CM考古]]` |
| 血液科/腫瘤科 | `[[Hema考古]]` |
| 風濕科/免疫科 | `[[AIR考古]]` |
| 感染科 | `[[Inf考古]]` |
| 神經科 | `[[Neuro考古]]` |
| 精神科 | `[[Psy考古]]` |
| 小兒科 | `[[Ped考古]]` |
| 皮膚科 | `[[Derma考古]]` |
| 家醫科 | `[[FM考古]]` |
| 急診科 | `[[EM考古]]` |
| 醫學倫理 | `[[Ethics考古]]` |

**外科系與醫學六專科（醫學六：麻醉、眼、耳鼻喉、婦產、復健）**
| 科別 | 標籤 |
|------|------|
| 麻醉科 | `[[Anes考古]]` |
| 眼科 | `[[Ophth考古]]` |
| 耳鼻喉科 | `[[ENT考古]]` |
| 婦產科 | `[[OBGYN考古]]` |
| 復健科 | `[[Rehab考古]]` |

**科別組合原則**：一題可有多個科別 tag。例如兒童心臟病 → `[[Ped考古]], [[CV考古]]`；兒童感染症 → `[[Ped考古]], [[Inf考古]]`；皮膚科自體免疫疾病 → `[[Derma考古]], [[AIR考古]]`。

使用者也可以自行修改這份對照表的標籤命名（例如改成 `[[心臟科考古]]` 等），只要保持「一個科別一個 [[]] 標籤」的格式即可。

---

## 特殊題型處理
- **圖片題**：題目後加 `（🖼️ 圖片題）`（題目中有「如附圖」、「如圖所示」、「如下圖」等字樣）
- **一律給分**：答案行寫 `一律給分（說明原因）`
- **複數答案**：如 `A 或 D（均給分）（說明）`
- **除空白均給分**：如 `除未作答者不給分外，其餘均給分（說明）`

---

## 完整範例

```
[[115-1-4]]Q1
母親帶 5 個月大的寶寶來接種卡介苗，下列衛教說明，何項最不適當？
(A) 提醒定期幫寶寶修剪指甲，避免抓傷注射部位
(B) 注射部位 1～2 週後會呈現一個小紅結節，可能會有些微痛癢
(C) 注射部位 4～6 週後可能會有膿瘍或潰爛，需擠壓引流出膿
(D) 注射部位平均 4 個月後會開始結痂，留下淡紅色的小疤痕 #flashcard 
C（BCG 接種後膿瘍或潰爛屬正常免疫反應，應保持乾燥清潔自然癒合，禁忌擠壓引流以免續發感染與形成大疤痕）[[BCG]], [[tuberculosis]], [[Ped考古]], [[Inf考古]]

---

[[114-2-3]]Q8
下列有關高血壓藥物治療的敘述，何者錯誤？
(A) 高血壓合併左心室功能異常的病人，選用利尿劑、angiotensin-converting enzyme inhibitor、angiotensin receptor blocker 或 β-blocker 皆可
(B) 高血壓合併蛋白尿的病人，宜優先選用 β-blocker 或利尿劑
(C) ＜50 歲的單純高血壓病人，可優先選用 angiotensin-converting enzyme inhibitor
(D) ＞80 歲的單純高血壓病人，可優先選用利尿劑及鈣離子阻斷劑 #flashcard 
A 或 B（均給分）（B 明確錯誤：高血壓合併蛋白尿應優先選用 ACEi 或 ARB，以保護腎臟）[[hypertension]], [[left ventricular dysfunction]], [[proteinuria]], [[diuretics]], [[ACEi]], [[ARB]], [[beta-blocker]], [[calcium channel blocker]], [[CV考古]]

---

[[114-2-3]]Q32
下列那一種腸道細菌的感染最不易誘發反應性關節炎（reactive arthritis）的發生？
(A) Shigella
(B) Yersinia
(C) Clostridium
(D) Campylobacter #flashcard 
C（反應性關節炎常見誘發菌包括 Shigella、Yersinia、Campylobacter、Salmonella；Clostridium 最不相關）[[reactive arthritis]], [[Shigella]], [[Yersinia]], [[Clostridium]], [[Campylobacter]], [[AIR考古]]
```

注意範例第一行**沒有** `TARGET DECK:`，直接從第一題開始。

---

## 工作流程總覽

1. **Step 0**：確認輸入檔案路徑、輸出資料夾路徑（沒給就問）
2. **Step 1**：辨識考卷代號（年份-次數-X）並向使用者確認
3. **Step 2**：問分批方式（80 / 40 / 20 / 10）
4. **Step 3**：依副檔名（PDF / RTF / DOCX / MD / TXT）用對應 parser 轉純文字並讀取
5. **Step 4**：解析答案 + 特殊備註（均給分、一律給分）
6. **Step 5**：分批生成 flashcard，每題逐一執行 tag scan checklist
7. **Step 6**：寫入使用者指定的 vault 資料夾下 `醫學{三/四/五/六}/<代號>/<代號>.md`
8. **驗證**：用 bash 確認題數、flashcard 標記數、分隔符、圖片題、答案首字逐一比對

### 驗證指令範本
```bash
F="<檔案絕對路徑>"
echo "===題數===" && grep -c "^\[\[<代號>\]\]Q" "$F"
echo "===flashcard===" && grep -c "#flashcard " "$F"
echo "===分隔符===" && grep -c "^---$" "$F"
echo "===圖片題===" && grep -c "圖片題" "$F"
echo "===答案首字===" && awk '/#flashcard / {getline; q++; print q": "substr($0,1,4)}' "$F"
```

---

### 修正模式
若使用者指出 tag 漏標或多標，依照 Checklist 重新逐字掃描指定題目並輸出完整更正版。

若使用者指出某幾題答案錯誤、或要把舊有檔案的 `TARGET DECK:` 行移除、或要把錯位資料夾搬到正確路徑，依指示修正即可。
