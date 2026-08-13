---
name: video-to-openscad
description: Convert 360-degree rotation videos, top-view videos, multi-angle images, and measured dimensions into modular parametric OpenSCAD models with frame extraction, feature decomposition, STRUCTURE/KEEP/VOID/CLEARANCE classification, dependency planning, collision and clearance checks, multi-view rendering, visual comparison, iterative correction, and SCAD/STL/optional 3MF export. Use for engineering-shaped objects such as enclosures, mounts, brackets, fixtures, containers, motor holders, PCB cases, and printable mechanical parts reconstructed from video or image references.
---

# Video-to-3D OpenSCAD Skill

## 目的

本 Skill 用於將 360° 旋轉影片、俯視影片、多角度影像與使用者提供的實際尺寸，自動轉換為可參數化、可修改、可驗證的 OpenSCAD 3D 模型。

本 Skill 的核心不是單純「看影像後直接生成一個 `.scad`」，而是要求 Agent 先進行：

環境檢查  
→ 影片檢查  
→ 影片抽幀  
→ 多視角分析  
→ 幾何特徵拆解  
→ 尺寸建立  
→ STRUCTURE / KEEP / VOID / CLEARANCE 分類  
→ Dependency Graph  
→ Ordered Feature Pipeline  
→ OpenSCAD Modules  
→ Collision Check  
→ CSG 順序建模  
→ Clearance Check  
→ Render  
→ 視覺比對  
→ 自動修正  
→ 輸出 SCAD / STL / 3MF

最終目標是產生：

- `model.scad`
- `model.stl`
- 可選 `model.3mf`
- 多角度 Render 預覽
- 幾何分析文件
- 尺寸與參數表
- Collision / Clearance 檢查報告

---

# 0. 啟動時環境檢查

每一個新專案第一次執行本 Skill 時，必須先執行 Environment Check。

在確認必要軟體可以正常使用之前，不得進入正式建模流程。

## 0.0 自動補齊缺少的環境

依序執行「偵測 → 安裝缺失項目 → 重新偵測 → 保存報告」，不得只回報缺失後繼續建模。

1. 偵測作業系統、CPU 架構、可用套件管理器、目前 Python 執行檔及權限範圍。
2. 先列出已安裝、缺少及即將安裝的 Required 項目。
3. 缺少 Python 套件時，優先建立專案本機 `.venv`，再以該環境的 Python 執行 `pip install opencv-python numpy PyYAML Pillow`。後續腳本固定使用同一個 Python 執行檔。
4. 缺少 OpenSCAD、FFmpeg、ffprobe 或 Python 時，使用作業系統現有且可信任的套件管理器安裝：Windows 優先 `winget`，macOS 優先 Homebrew，Debian/Ubuntu 優先 `apt`，Fedora/RHEL 優先 `dnf`，Arch 優先 `pacman`。
5. 執行安裝前先顯示實際指令。若安裝需要系統管理員權限、接受授權條款、修改系統範圍或目前執行環境要求核准，先取得使用者確認；不得繞過權限機制。
6. 不使用來源不明的下載網址、任意遠端安裝腳本或未驗證的二進位檔。套件管理器無法使用時，標記 `BLOCKED` 並提供官方安裝方式。
7. 安裝完成後重新執行所有版本與 import 檢查。只有全部 Required 項目通過才可標記 `READY` 並開始建模。
8. 將偵測結果、執行過的安裝指令、版本、Python 路徑及失敗原因寫入 `analysis/environment_report.md`；不得記錄密碼、token 或其他憑證。

不得因快取顯示先前成功就忽略本次指令不存在或 import 失敗。自動安裝失敗時最多針對明確原因修正並重試一次，之後停止並回報，不得反覆安裝。

## 0.1 必要軟體

### OpenSCAD

用途：

- 解析 `.scad`
- Render
- CSG 運算
- STL 匯出
- 最終模型驗證

檢查：

```bash
openscad --version
```

若指令不存在：

```text
ERROR: OpenSCAD not found
STATUS: BLOCKED
```

此時先依 0.0 的自動補齊流程安裝 OpenSCAD，重新檢查仍失敗才停止建模。

---

### FFmpeg

用途：

- 讀取影片
- 取得影片資訊
- 抽取影格
- 指定時間點截圖
- 建立多視角分析資料

檢查：

```bash
ffmpeg -version
ffprobe -version
```

`ffprobe` 應用於取得：

- Resolution
- FPS
- Duration
- Codec
- Frame Count
- Rotation Metadata

若缺少 FFmpeg：

```text
ERROR: FFmpeg not found
STATUS: BLOCKED
```

此時先依 0.0 的自動補齊流程安裝 FFmpeg（`ffprobe` 通常由同一套件提供），重新檢查仍失敗才停止建模。

---

### Python

用途：

- 影片處理輔助
- 影像資料整理
- 自動化腳本
- Render 控制
- 尺寸資料處理
- JSON / YAML 處理
- 驗證流程

依序嘗試：

```bash
python --version
python3 --version
```

Windows 可另外嘗試：

```powershell
py --version
```

---

## 0.2 Python 必要套件

至少確認：

- `opencv-python`
- `numpy`
- `PyYAML`
- `Pillow`

檢查範例：

```bash
python -c "import cv2, numpy, yaml, PIL; print('Python dependencies OK')"
```

若缺少套件，必須列出缺少項目。

例如：

```text
Missing Python packages:
- opencv-python
- PyYAML

STATUS: BLOCKED
```

缺少時先依 0.0 建立或重用專案 `.venv` 並安裝：

```bash
python -m pip install opencv-python numpy PyYAML Pillow
```

---

## 0.3 建議軟體

### Git

用途：

- 模型版本控制
- Codex 修改追蹤
- Feature 修改紀錄
- 回復上一版本

檢查：

```bash
git --version
```

缺少時：

```text
WARNING: Git not found
```

不阻止建模流程。

### ImageMagick

用途：

- 圖片轉換
- Contact Sheet
- Render 比較
- 多圖組合

檢查：

```bash
magick -version
```

缺少時只標記為 Optional。

---

## 0.4 Required / Optional 分級

### Required

- OpenSCAD
- FFmpeg
- ffprobe
- Python
- numpy
- opencv-python
- PyYAML
- Pillow

任一 Required 缺少時：

```text
STATUS: BLOCKED
```

不得進入正式建模。

### Optional

- Git
- ImageMagick

缺少時：

```text
STATUS: WARNING
```

但流程可以繼續。

---

## 0.5 Environment Report

第一次執行必須建立：

```text
analysis/environment_report.md
```

範例：

```text
# Environment Check

OpenSCAD
Status: OK
Version: detected

FFmpeg
Status: OK

ffprobe
Status: OK

Python
Status: OK

OpenCV
Status: OK

NumPy
Status: OK

PyYAML
Status: OK

Pillow
Status: OK

Git
Status: OK

ImageMagick
Status: OPTIONAL / NOT FOUND

Overall Status:
READY
```

---

## 0.6 環境快取

第一次檢查成功後可建立：

```text
.project/environment.json
```

例如：

```json
{
  "checked": true,
  "openscad": true,
  "ffmpeg": true,
  "ffprobe": true,
  "python": true,
  "opencv": true,
  "numpy": true,
  "pyyaml": true,
  "pillow": true
}
```

同一專案後續執行時，不需每一步重新完整檢查。

但若執行 CLI 時發生：

```text
command not found
```

或環境明顯改變，必須重新執行 Environment Check。

---

# 1. 適用範圍

本 Skill 優先適用於具有明確工程幾何特徵的物體，例如：

- 3D 列印零件
- 電子設備外殼
- PCB 外殼
- 感測器盒
- 馬達固定座
- 支架
- 轉接座
- 機械零件
- 旋鈕
- 治具
- 固定架
- 教具
- 容器
- 工具外殼

特別適合：

- cube
- cylinder
- sphere
- prism
- polygon extrusion
- rotational body
- slot
- hole
- boss
- rib
- shell
- bracket
- mount
- cutout

不適合作為主要建模方法：

- 人體
- 動物
- 雕塑
- 布料
- 植物
- 高度不規則自由曲面
- 有機造型
- 複雜藝術曲面

上述類型應優先考慮 Mesh / Photogrammetry / 3D Scan。

---

# 2. 輸入資料

## 2.1 影片與影像

推薦提供：

- 360° 水平旋轉影片
- 斜上方旋轉影片
- 俯視
- 底視
- 關鍵孔位特寫
- 接頭位置特寫

至少應能取得：

- 正面
- 側面
- 背面
- 俯視

等主要幾何資訊。

---

## 2.2 尺寸資料

推薦使用：

```text
input/dimensions.yaml
```

至少提供：

- overall_width
- overall_depth
- overall_height

若有工程特徵，盡量提供：

- hole_diameter
- hole_spacing
- wall_thickness
- slot_width
- slot_height
- boss_diameter
- boss_height
- screw_diameter
- PCB_width
- PCB_length
- PCB_height
- motor_size
- battery_size

範例：

```yaml
unit: mm

overall:
  width: 80
  depth: 50
  height: 40

shell:
  wall_thickness: 2.4

pcb:
  width: 70
  depth: 40
  height: 1.6

holes:
  main_diameter: 12
  mount_diameter: 4
  mount_spacing: 60
```

---

# 3. Input Check

環境正常後才檢查輸入資料。

推薦目錄：

```text
input/
├─ object.mp4
└─ dimensions.yaml
```

影片格式可包含：

- `.mp4`
- `.mov`
- `.mkv`
- `.avi`

若缺少必要輸入，停止建模並列出缺少項目。

---

# 4. Video Inspection

抽幀前先使用 `ffprobe` 分析影片。

取得：

- Resolution
- FPS
- Duration
- Codec
- Rotation Metadata
- Frame Count

範例：

```bash
ffprobe -v error \
-show_entries stream=width,height,r_frame_rate,codec_name \
-show_entries format=duration \
-of json \
input/object.mp4
```

結果保存：

```text
analysis/video_info.json
```

---

# 5. Video Quality Check

Agent 必須先檢查影片是否適合幾何辨識：

- 物體是否完整出現在畫面
- 是否過度模糊
- 背景是否干擾
- 曝光是否正常
- 旋轉是否連續
- 是否具有足夠側面資訊
- 是否具有俯視資訊
- 是否具有比例參考
- 是否有遮擋

影片品質不足時，不得宣稱可以得到精確工程尺寸。

---

# 6. Frame Extraction

不需要分析影片中的全部影格。

第一輪推薦抽取 8～16 張代表影格：

```text
frames/
├─ frame_000_front.png
├─ frame_045.png
├─ frame_090_right.png
├─ frame_135.png
├─ frame_180_back.png
├─ frame_225.png
├─ frame_270_left.png
├─ frame_315.png
└─ frame_top.png
```

若幾何不足，再補抽局部角度。

---

# 7. Geometry Analysis

Codex 從多角度影像辨識：

- 主要外輪廓
- 底座
- 外殼
- 柱體
- 凸台
- 孔
- 槽
- 支架
- 加強肋
- 接頭開孔
- 固定孔
- 曲面
- 旋轉體

建立：

```text
analysis/geometry.md
```

---

# 8. Feature Decomposition

禁止未分析結構就直接生成最終 SCAD。

模型必須先拆成 Feature。

例如：

```text
Final Model
│
├─ STRUCTURE
│  ├─ shell
│  ├─ base
│  └─ bracket
│
├─ KEEP
│  ├─ pcb_standoff
│  ├─ motor_mount
│  └─ reinforcement
│
├─ VOID
│  ├─ pcb_keepout
│  ├─ motor_space
│  └─ battery_space
│
└─ CLEARANCE
   ├─ connector_clearance
   ├─ cable_clearance
   └─ assembly_clearance
```

原則：

一個明確工程特徵，原則上對應一個 OpenSCAD `module()`。

---

# 9. 四大幾何類型

## STRUCTURE

真正要列印並永久保留的主要實體。

例如：

- 外殼
- 底板
- 支架
- 外部凸台
- 外框

## KEEP

即使進行內部挖空仍必須保留的內部結構。

例如：

- PCB 螺絲柱
- PCB 固定座
- 馬達座
- 電池架
- 肋條
- 定位柱
- 螺帽座

KEEP 不可因 cavity subtraction 被誤刪。

## VOID

一定必須被挖除的設備或功能空間。

例如：

- PCB 空間
- 馬達本體空間
- 電池空間
- 主要中空腔體
- 接頭本體空間

## CLEARANCE

為安裝、移動、插拔、公差、線材所需要的額外空間。

例如：

- USB 插拔空間
- JST 接頭拔插空間
- 線材彎曲空間
- PCB 安裝公差
- 馬達振動間隙
- 工具操作空間
- 電池取出空間

VOID 與 CLEARANCE 不可混為一談。

---

# 10. 設備 Envelope / Keep-out Volume

若容器內有機電設備，必須建立設備 Envelope。

例如 PCB 實體：

```text
70 × 40 × 1.6 mm
```

但設備真正需要的 Keep-out Volume 可能是：

```text
74 × 44 × 12 mm
```

因為必須包含：

- 元件高度
- 排針
- USB
- JST
- 焊點
- 排線
- 安裝公差
- 拔插空間

OpenSCAD 示例：

```scad
module pcb_keepout() {
    cube([74, 44, 12], center=true);
}
```

`pcb_keepout()` 不是 PCB 本體，而是禁止其他結構侵入的空間。

---

# 11. 尺寸優先權

尺寸來源依下列順序採用：

```text
使用者實測尺寸
>
已知標準零件尺寸
>
影像比例估算
>
AI 推測
```

非實測尺寸必須標示：

```text
estimated
```

禁止將估計值描述為精確實測值。

---

# 12. Dependency Graph

生成 OpenSCAD 前必須先建立：

```text
analysis/dependency_graph.md
```

每個 Feature 都要知道：

- 依賴哪些 Feature
- 必須在哪些 Feature 之前
- 必須在哪些 Feature 之後
- 是否可被 VOID 切除
- 是否必須保留

---

# 13. Ordered Feature Pipeline

建模順序不可任意交換。

推薦順序：

```text
01 外部基準體
↓
02 主要 Shell
↓
03 外部結構
↓
04 內部 KEEP
↓
05 Collision Check
↓
06 設備 Envelope
↓
07 主要 VOID
↓
08 功能 Ports
↓
09 Fasteners
↓
10 CLEARANCE
↓
11 上蓋 / 下殼分割
↓
12 Collision Check
↓
13 Clearance Check
↓
14 Render
↓
15 Export
```

---

# 14. OpenSCAD 建模數學概念

基本模型可表示為：

$$
M=
\left(
S_{\text{shell}}
-
V_{\text{cavity}}
\right)
\cup
F_{\text{internal}}
$$

其中：

- $S_{\text{shell}}$：外殼
- $V_{\text{cavity}}$：主要內部空間
- $F_{\text{internal}}$：必須保留的內部結構

最後：

$$
M_{\text{final}}
=
M
-
V_{\text{ports}}
-
V_{\text{holes}}
-
V_{\text{clearance}}
$$

KEEP 的處理順序尤其重要。

---

# 15. OpenSCAD Module Generation

推薦依功能分類：

```text
modules/
├─ structure/
├─ keep/
├─ void/
├─ clearance/
├─ ports/
└─ fasteners/
```

禁止把所有幾何塞入單一巨大 module。

---

# 16. OpenSCAD Pipeline

推薦採用明確 Stage。

```scad
module stage_01_structure() {
    union() {
        outer_body();
        external_brackets();
    }
}

module stage_02_shell() {
    difference() {
        stage_01_structure();
        inner_cavity();
    }
}

module stage_03_internal_keep() {
    union() {
        stage_02_shell();
        pcb_standoffs();
        motor_mount();
        battery_mount();
        reinforcement_ribs();
    }
}

module stage_04_device_voids() {
    difference() {
        stage_03_internal_keep();
        pcb_keepout();
        motor_keepout();
        battery_keepout();
    }
}

module stage_05_ports() {
    difference() {
        stage_04_device_voids();
        usb_port();
        switch_port();
        cable_exit();
    }
}

module stage_06_fasteners() {
    difference() {
        stage_05_ports();
        screw_holes();
        countersinks();
        nut_traps();
    }
}

module final_model() {
    stage_06_fasteners();
}

final_model();
```

---

# 17. Collision Check

每個主要 Stage 完成後，檢查不應互相重疊的幾何。

概念：

$$
V(A\cap B)
$$

若：

$$
V(A\cap B)>0
$$

且 A、B 不允許重疊，即為 Collision。

常見 Collision：

- PCB 與外殼內壁
- PCB 與螺絲柱
- PCB 與馬達
- 電池與支架
- 馬達與外殼
- USB 與外殼
- 電線與加強肋
- 螺絲與內部設備

發生衝突時：

```text
STOP
↓
定位 Feature
↓
修正 Module
↓
重新 Render
```

不可直接忽略。

---

# 18. Collision 修正順序

發現衝突後依序考慮：

1. 移動內部結構
2. 改變支架尺寸
3. 改變設備位置
4. 改變外殼尺寸
5. 改變 KEEP
6. 改變 VOID
7. 改變 CLEARANCE
8. 最後才考慮刪除結構

禁止為了消除 Collision 任意刪除重要 Feature。

---

# 19. Clearance Check

Collision 與 Clearance 必須分開。

Collision：已經互相穿透。

Clearance：沒有相交，但距離不足。

因此必須另外進行 Clearance Check。

---

# 20. 參數化規則

禁止大量硬編碼 Magic Numbers。

重要尺寸必須集中管理，推薦儲存在：

```text
config/dimensions.scad
```

---

# 21. Render Validation

完成 SCAD 後不得立即視為完成。

必須使用 OpenSCAD Render。

推薦輸出：

- front
- rear
- left
- right
- top
- bottom
- isometric
- alternate isometric

並與原始影片代表影格進行視覺比較。

---

# 22. 驗證項目

每次 Render 後至少檢查：

- 外輪廓
- 長寬高比例
- 孔位
- 孔徑
- 槽位置
- 凸台位置
- 外殼厚度
- PCB 空間
- 馬達空間
- 電池空間
- 螺絲柱
- CLEARANCE
- Collision

---

# 23. Iteration Loop

```text
Analyze
↓
Generate
↓
Render
↓
Compare
↓
Collision Check
↓
Clearance Check
↓
Pass?
```

若不通過：

```text
定位錯誤 Feature
↓
只修改相關 Module
↓
重新 Render
```

禁止因為局部 Feature 錯誤而重寫整個模型。

---

# 24. 建議專案結構

```text
video-to-3d-openscad/
│
├─ SKILL.md
├─ README.md
│
├─ input/
│  ├─ object.mp4
│  └─ dimensions.yaml
│
├─ frames/
│
├─ analysis/
│  ├─ environment_report.md
│  ├─ video_info.json
│  ├─ geometry.md
│  ├─ feature_tree.md
│  ├─ dependency_graph.md
│  ├─ collision_report.md
│  └─ clearance_report.md
│
├─ config/
│  └─ dimensions.scad
│
├─ modules/
│  ├─ structure/
│  ├─ keep/
│  ├─ void/
│  ├─ clearance/
│  ├─ ports/
│  └─ fasteners/
│
├─ scripts/
│  ├─ check_environment.py
│  ├─ inspect_video.py
│  ├─ extract_frames.py
│  ├─ render.py
│  └─ validate.py
│
├─ render/
│
├─ output/
│
└─ .project/
   └─ environment.json
```

---

# 25. 最終輸出

成功完成後至少輸出：

```text
output/
├─ model.scad
├─ model.stl
└─ preview/
```

若流程支援，可另外輸出：

```text
model.3mf
```

---

# 26. Definition of Done

只有滿足以下條件才能視為完成：

- Environment Check 已通過
- 已完成影片基本資訊分析
- 已完成代表影格擷取
- 已完成 Feature Decomposition
- 已建立參數化尺寸
- 已分類 STRUCTURE / KEEP / VOID / CLEARANCE
- 已建立設備 Envelope
- 已建立 Dependency Graph
- 已決定 Ordered Feature Pipeline
- 已產生獨立 OpenSCAD Modules
- 已正確執行 CSG 順序組合
- 已完成 Collision Check
- 已完成 Clearance Check
- 已完成多角度 Render
- 已與原影片影格進行視覺比較
- 已修正主要差異
- 已成功匯出 STL
- `model.scad` 可重新修改參數並重新生成

---

# 27. 禁止事項

Agent 不得：

- 未做 Environment Check 就開始建模
- 未分析結構就直接生成最終 SCAD
- 將所有幾何塞入一個巨大 module
- 任意改變 Feature 建模順序
- 將 KEEP 誤當 VOID
- 為了消除 Collision 直接刪除重要結構
- 忽略設備 Envelope
- 忽略 CLEARANCE
- 使用大量未命名 Magic Numbers
- 將估計尺寸描述為實測尺寸
- 未 Render 就聲稱模型完成
- 未做 Collision Check 就匯出最終模型

---

# 28. Agent 核心角色

本 Skill 不將 Codex 定義為單純的 OpenSCAD 程式產生器。

Codex 應扮演：

```text
Visual Geometry Analyst
        +
Feature Decomposition Planner
        +
CAD Modeling Planner
        +
Dependency Manager
        +
CSG Pipeline Planner
        +
Collision / Clearance Inspector
        +
OpenSCAD Code Generator
```

核心流程：

```text
環境
↓
影片
↓
理解
↓
拆解
↓
分類
↓
排序
↓
建模
↓
檢查
↓
驗證
↓
修正
↓
輸出
```

---

# 29. 最重要規則

1. 第一次執行必須先檢查 OpenSCAD、FFmpeg、ffprobe、Python 與必要 Python 套件。
2. 先理解物件，再寫 OpenSCAD。
3. 先拆 Feature，再決定 Module。
4. 先建立 Dependency Graph，再決定建模順序。
5. STRUCTURE、KEEP、VOID、CLEARANCE 必須分開管理。
6. 容器內有機電設備時，必須建立設備 Envelope / Keep-out Volume。
7. VOID 與 CLEARANCE 必須在正確階段扣除。
8. KEEP 不可被內部挖空流程誤刪。
9. 每個主要 Feature 原則上應獨立成 Module。
10. 每次修改優先修改單一 Feature。
11. Collision Check 與 Clearance Check 必須分開。
12. OpenSCAD Render 是必要驗證步驟。
13. 未完成驗證，不得宣告模型完成。
14. 最終模型必須保持參數化、可修改、可重新生成。
