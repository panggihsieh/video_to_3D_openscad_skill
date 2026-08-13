# Video to 3D OpenSCAD Skill

這個專案的目標是把 360° 旋轉影片、多角度影像與已知尺寸，轉成可參數化、可驗證、可重建的 OpenSCAD 模型。

專案網址：[panggihsieh/video_to_3D_openscad_skill](https://github.com/panggihsieh/video_to_3D_openscad_skill)

本倉庫提供一份給 AI Agent 使用的完整建模流程規格。主要文件 [`SKILL.md`](SKILL.md) 定義環境檢查、影片抽幀、幾何特徵拆解、尺寸來源優先權、OpenSCAD 模組化建模、碰撞與間隙檢查，以及多視角渲染驗證等規則。它適合用來協助重建外殼、支架、固定座、治具及其他具有明確工程幾何特徵的物件。

對應輸出包含：
- `model.scad`
- `model.stl`
- 可選 `model.3mf`
- 多角度 render 預覽
- 幾何分析與檢查報告

## 取得與使用

下載專案：

```bash
git clone https://github.com/panggihsieh/video_to_3D_openscad_skill.git
cd video_to_3D_openscad_skill
```

使用時，將 `SKILL.md` 提供給支援自訂指令或 Skill 的 AI Agent，並一併提供：

- 物件的 360° 旋轉影片或多角度照片
- 至少一組可靠的實測尺寸，建議包含整體寬、深、高
- 如有需要，再補充孔徑、孔距、壁厚及設備預留空間等工程尺寸

範例提示詞：

```text
請依照 SKILL.md 的流程，分析 input/object.mp4 與
input/dimensions.yaml，建立參數化 OpenSCAD 模型。先完成環境與輸入檢查，
再依序進行抽幀、特徵拆解、碰撞／間隙檢查和多視角渲染驗證。
```

這套流程著重工程幾何重建，不是攝影測量工具。影像無法可靠提供絕對尺寸，因此關鍵尺寸仍應以實際量測值為準；由影像估算或 AI 推測的數值必須標記為 `estimated`。

## Codex Skill 使用說明

本專案是可安裝的 Codex Skill。完整指令位於 GitHub 倉庫中的 [`SKILL.md`](https://github.com/panggihsieh/video_to_3D_openscad_skill/blob/main/SKILL.md)。

### 方法一：直接從 GitHub 使用

不安裝 Skill 時，可在 Codex 對話中提供上述 GitHub 文件網址，或先 Clone 倉庫，再要求 Codex 讀取本機文件：

```text
請讀取 SKILL.md，並依照其中的完整流程處理這次建模工作。
輸入影片：input/object.mp4
尺寸資料：input/dimensions.yaml
請先執行環境檢查，不要跳過 Collision Check、Clearance Check 與 Render Validation。
```

### 方法二：從 GitHub 安裝 Skill（建議）

在另一台已安裝 Codex 的電腦開啟對話，輸入：

```text
使用 $skill-installer，從以下 GitHub 倉庫安裝 Skill：
https://github.com/panggihsieh/video_to_3D_openscad_skill
```

安裝完成後重新開啟 Codex 或建立新對話，再於 Codex CLI 或 IDE extension 使用 `/skills` 確認 `video-to-openscad` 已出現在清單中。之後可在提示詞中以 `$` 明確指定：

```text
使用 $video-to-openscad 分析 input/object.mp4 與 input/dimensions.yaml，
建立可參數化的 OpenSCAD 模型，完成多視角渲染與裝配間隙驗證後再匯出 STL。
```

也可以直接描述工作內容。當請求與 Skill 的適用範圍相符時，Codex 可依 Skill 的 `description` 自動判斷是否使用；若希望確保本次一定套用，建議明確寫出 `$video-to-openscad`。

> [!IMPORTANT]
> 安裝 Skill 只會提供建模流程與規則，不會自動安裝 OpenSCAD、FFmpeg、Python 或 Python 套件。第一次建模時仍必須完成環境檢查。

Codex Skill 的結構與啟用方式可參考 [OpenAI 官方 Build skills 文件](https://learn.chatgpt.com/codex/build-skills)。

### 建議提供給 Skill 的資料

- 360° 旋轉影片、俯視影片或足夠的多角度照片
- 整體寬、深、高等實測基準尺寸
- 孔徑、孔距、壁厚、槽寬、螺絲與設備尺寸
- 物件用途、裝配方向及需要保留的結構
- 期望輸出格式，例如 `.scad`、`.stl` 或 `.3mf`

### 完整呼叫範例

```text
使用 $video-to-openscad 處理 input/object.mp4。

已知尺寸位於 input/dimensions.yaml，單位為 mm。請先確認 OpenSCAD、FFmpeg、
ffprobe、Python 與必要套件可用，再抽取代表影格並建立幾何分析、Feature Tree、
Dependency Graph 和尺寸參數。請將特徵分成 STRUCTURE、KEEP、VOID、CLEARANCE，
用獨立 modules 建模。完成 Collision Check、Clearance Check 和多視角 Render 比對後，
輸出 output/model.scad、output/model.stl 與預覽圖。所有推估尺寸標記為 estimated。
```

## 使用重點

1. **先確認輸入品質**：物件應完整入鏡、輪廓清晰、光線穩定，並盡量涵蓋正面、背面、左右側、頂部及底部。
2. **至少提供一組實測尺寸**：建議提供整體寬、深、高；孔徑、孔距、壁厚及裝配尺寸越完整，模型越可靠。
3. **先分析再建模**：先抽取代表影格、辨識幾何特徵並建立尺寸表，不要直接由單張影像生成最終模型。
4. **依功能拆分 Feature**：將幾何分成 `STRUCTURE`、`KEEP`、`VOID`、`CLEARANCE`，並讓主要工程特徵各自對應獨立的 OpenSCAD `module()`。
5. **保持參數化**：將重要尺寸集中存放於參數檔，避免在不同模組中散落未命名的固定數值。
6. **逐階段驗證**：每完成一個主要建模階段，就檢查碰撞、裝配間隙及 CSG 運算順序。
7. **完成多視角比對**：輸出前至少檢查正面、背面、左右側、頂部、底部及等角視圖，並與原始影像比較。

## 注意事項

- 本 Skill 適合外殼、支架、固定座、治具及容器等工程造型，不適合人體、動物、布料或複雜有機曲面；這些物件應優先考慮攝影測量或 3D 掃描。
- 影像只能用來推估比例與形狀，不能取代卡尺等實際量測工具。所有非實測尺寸都必須標記為 `estimated`。
- 拍攝時若使用廣角鏡頭、距離過近、物件遭遮擋或缺少比例參考，會增加透視變形與尺寸判讀誤差。
- `KEEP` 是必須保留的結構，`VOID` 是設備本體所需空間，`CLEARANCE` 是安裝、插拔、公差及線材活動所需的額外空間，三者不可混用。
- 沒有幾何穿透不代表能正常裝配；Collision Check 與 Clearance Check 必須分開執行。
- 未完成 OpenSCAD Render、多視角比對及 STL 匯出驗證前，不應將模型視為完成。
- 產生的模型仍應在製造前由使用者確認尺寸、壁厚、公差、材料收縮、螺絲規格及實際裝配安全性。

## 工作流程

以下流程建議固定順序執行，不要跳步：

1. 環境檢查（第一次執行必做）
2. 輸入檢查（影片 + 尺寸）
3. 影片資訊分析（ffprobe）
4. 影片品質檢查
5. 抽取代表影格
6. 多視角幾何分析
7. Feature 拆解
8. STRUCTURE / KEEP / VOID / CLEARANCE 分類
9. 建立 Dependency Graph
10. 決定 Ordered Feature Pipeline
11. 產生 OpenSCAD modules（分功能）
12. 分 stage 組合 CSG
13. Collision Check
14. Clearance Check
15. 多角度 Render 驗證
16. 視覺比對與迭代修正
17. 匯出 SCAD / STL / 3MF（選用）

## 快速上手

### 1) 建議目錄結構

```text
video-to-3d-openscad/
├─ input/
│  ├─ object.mp4
│  └─ dimensions.yaml
├─ frames/
├─ analysis/
├─ config/
├─ modules/
├─ scripts/
├─ render/
├─ output/
└─ .project/
```

### 2) 環境檢查（Required）

請先確認以下工具可用：
- OpenSCAD
- FFmpeg
- ffprobe
- Python
- Python 套件：opencv-python、numpy、PyYAML、Pillow

範例檢查指令：

```bash
openscad --version
ffmpeg -version
ffprobe -version
python --version
python -c "import cv2, numpy, yaml, PIL; print('Python dependencies OK')"
```

如果 Required 有任一缺少，流程狀態應視為 `BLOCKED`，先補齊再建模。

### 3) 輸入資料準備

- 影片：建議提供 360° 旋轉或至少前/後/左/右/俯視資訊。
- 尺寸：至少提供 overall width/depth/height。
- 工程特徵（建議）：孔徑、孔距、壁厚、槽寬、柱體尺寸等。

`input/dimensions.yaml` 範例：

```yaml
unit: mm

overall:
  width: 80
  depth: 50
  height: 40

shell:
  wall_thickness: 2.4

holes:
  mount_diameter: 4
  mount_spacing: 60
```

### 4) 影片分析與抽幀

先做影片資訊分析，再抽取 8 到 16 張代表影格（front/right/back/left/top 等）。

建議保存：
- `analysis/video_info.json`
- `frames/frame_*.png`

### 5) Feature-first 建模

請先做「理解與拆解」，不要直接寫最終模型。

最小拆解原則：
- 一個明確工程特徵，原則上一個 `module()`
- 先定義依賴，再決定建模順序
- 先建 STRUCTURE，再加入 KEEP，再做 VOID 與 CLEARANCE

### 6) 分階段組裝（CSG Pipeline）

建議使用 stage pipeline：
- stage_01：外部基準與主結構
- stage_02：shell 與 cavity
- stage_03：內部 KEEP
- stage_04：設備 VOID
- stage_05：ports
- stage_06：fasteners
- final_model：最終輸出

### 7) 驗證與修正

每個主要 stage 後都要做：
- Collision Check（是否已穿透）
- Clearance Check（距離是否不足）
- Render 視覺比對（外形、比例、孔位、槽位、設備空間）

不通過時僅修正對應 feature module，避免重寫整體模型。

### 8) 輸出

至少輸出：
- `output/model.scad`
- `output/model.stl`
- `output/preview/`

可選輸出：
- `output/model.3mf`

## 使用注意事項

### 1. 不可跳過環境檢查

第一次執行專案時，必須先做環境檢查並保存報告。未通過時不得進入建模。

### 2. 尺寸優先權必須明確

尺寸來源請依序採用：
1. 使用者實測尺寸
2. 標準零件尺寸
3. 影像比例估算
4. AI 推測

凡非實測尺寸，請明確標註 `estimated`。

### 3. KEEP 與 VOID 不可混用

- KEEP：內部必須保留的結構（例如螺絲柱、支撐肋）
- VOID：必須挖除的功能空間（例如 PCB 或馬達空間）

錯誤混用會造成結構被誤刪或設備無法裝配。

### 4. VOID 與 CLEARANCE 必須分開

- VOID：設備本體需要空間
- CLEARANCE：裝配、插拔、公差、線材彎折空間

只有 VOID 沒有 CLEARANCE，常見結果是「模型可列印但無法裝配」。

### 5. 先做 Dependency Graph 再寫模組

建議先定義每個 feature 的前後依賴、可否被切除、是否必須保留，再開始撰寫 `module()`。

### 6. 不要把所有幾何塞進單一巨大 module

請採功能分層。這樣才能：
- 針對單一 feature 快速修正
- 避免 CSG 順序錯誤
- 降低迭代成本

### 7. 衝突修正有優先順序

發現 collision 時，建議依序調整：
1. 內部結構位置
2. 支架尺寸
3. 設備位置
4. 外殼尺寸
5. KEEP / VOID / CLEARANCE 參數

不要用刪除重要結構當作第一解法。

### 8. 避免 Magic Numbers

重要尺寸請集中到參數檔（例如 `config/dimensions.scad`），不要散落在各 module 內。

### 9. Render 是必要條件，不是可選

未經多視角 render 與比對，不應宣告完成。

### 10. 定義完成標準（Definition of Done）

至少要同時滿足：
- 環境檢查通過
- 影片分析與抽幀完成
- Feature 分解與分類完成
- Dependency Graph 與 Pipeline 完成
- Collision / Clearance 檢查通過
- 可輸出並重建 `model.scad` 與 `model.stl`

## 建議維護方式

- 每次修改只動單一 feature 優先
- 變更記錄包含：改了哪個 module、為何改、對應哪個檢查結果
- 重要參數異動同步更新分析文件與 render 預覽
