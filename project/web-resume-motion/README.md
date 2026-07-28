# Web Resume — Motion

捲動驅動的單頁作品集。整頁只有一支影片，捲動時逐格 seek，同時控制影片的縮放、位置、左側裁切開口，以及頁面底色與文字色的漸變。

**線上版：** https://hanjason0926-lgtm.github.io/project/web-resume-motion/

---

## 檔案結構

```
web-resume-motion/
├── index.html          單一檔案：HTML + CSS + JS 全部內嵌
├── README.md           本文件
├── assets/
│   ├── scroll.mp4      主影片，15.4 MB（橫式 / 桌機）
│   ├── scroll-sm.mp4   輕量版，5.4 MB（直式 / 小螢幕 / 省流量模式）
│   ├── poster.jpg      封面圖，同時是影片載入失敗的降級底圖
│   └── lenis.min.js    平滑捲動函式庫
└── raw/                原始素材，已被 .gitignore 排除，不會上傳
```

## 運作原理

1. **捲動軌道** — `#track` 高 `640vh`，內層 `.pin` 用 `position:sticky` 釘住整個視窗。捲動距離換算成 `progress`（0 → 1）。
2. **影片逐格定位** — 影片以全 I-frame 編碼，所以 `currentTime` / `fastSeek()` 可以瞬間跳到任意格。`pump()` 用 `busy` 旗標避免 seek 排隊塞車，並附 220 ms 逾時保險。
3. **分鏡表** — `KEYS_LAND`（橫式）與 `KEYS_PORT`（直式）定義關鍵格：在某個 progress 時，影片要放多大、放哪裡、左邊切掉多少。中間值用 easeInOutCubic 內插。
4. **裁切開口** — `#stage` 的 `clip-path` 把影片左緣切掉，露出的區域就是「開口」。`cut` 值控制切在影片畫面的哪個比例，用途是遮掉影片本身內建的文字，讓 HTML 文字取而代之。
5. **色調同步** — `TONE` 表把頁面背景與文字色跟著影片明暗一起變。明暗翻轉的瞬間對比會塌掉，所以 `HUDO` 表讓 HUD 在那個區間先熄燈再亮起。
6. **效能** — 每幀都會算，但透過 `memo` 比對，只有值真的改變才寫回 DOM。

## 已知限制

- `scroll.mp4` 15.4 MB，網路較慢時 loader 會轉一陣子。載入邏輯是「緩衝超過 35% 或 `readyState>=3` 就放行」，並有 36 秒逾時保底。
- 影片必須是**全 I-frame 編碼**，否則逐格 seek 會卡頓。重新壓片時要保留這個特性（`-g 1`）。
- iOS 低耗電模式可能阻擋 `preload`，此時會 fallback 到 `poster.jpg` 靜態底圖。

---

## 單 Prompt 重建

把下面整段複製貼給 AI，就會產出一份與線上版功能相同的 `index.html`。影片與封面圖直接引用 GitHub Pages 上的網址，不需要下載任何素材，產出的單一 HTML 檔可以放在任何地方直接開啟。

```
請產生一個**單一檔案** `index.html`，包含完整 HTML + 內嵌 CSS + 內嵌 JS，不使用任何建置工具、框架或打包器。所有素材以下列絕對網址引用，**不要下載、不要改成相對路徑**：

- 主影片（橫式／桌機）：https://hanjason0926-lgtm.github.io/project/web-resume-motion/assets/scroll.mp4
- 輕量影片（直式／小螢幕／省流量）：https://hanjason0926-lgtm.github.io/project/web-resume-motion/assets/scroll-sm.mp4
- 封面圖／降級底圖：https://hanjason0926-lgtm.github.io/project/web-resume-motion/assets/poster.jpg
- 平滑捲動函式庫：https://unpkg.com/lenis@1.1.13/dist/lenis.min.js （用一般 script 標籤載入，全域變數為 Lenis）

影片標籤**不要加 crossorigin 屬性**（不需要 CORS，加了反而可能載入失敗）。該來源支援 HTTP Range，跨網域逐格 seek 可正常運作。

## 一、這是什麼

一個捲動驅動（scroll-driven）的單頁作品集。整頁核心只有一支影片：使用者捲動時，影片逐格 seek，同時它的縮放、位置、左側裁切開口、以及整個頁面的背景色與文字色都跟著連續變化。文字分成四個章節，在特定捲動區間淡入淡出。

影片是全 I-frame 編碼，所以 currentTime / fastSeek() 可以瞬間跳到任一格，不會卡頓。

## 二、版面骨架

- #track：高度 640vh，是捲動距離的來源。
- .pin：#track 的子層，position:sticky; top:0; height:100svh; overflow:hidden; isolation:isolate。所有視覺內容都在這裡面。
- 捲動進度 progress = clamp(-track.getBoundingClientRect().top / (track.offsetHeight - 視窗高), 0, 1)，值域 0 → 1，全站所有動畫都由它驅動。
- #stage（position:absolute; inset:0; z-index:1）內含 #film，#film 內含 <video id="vid" muted playsinline preload="auto" disablepictureinpicture>。
- #film 基準尺寸固定寫 width:834px; height:1112px; transform-origin:0 0，實際大小一律用 transform: translate3d(x,y,0) scale(w/834, h/1112) 控制。影片 object-fit:fill。
- 影片長寬比 ASPECT = 834/1112；FPS = 24；總格數 FRAMES = 361；DURATION = FRAMES/FPS。

## 三、分鏡表（核心，數值請完全照抄）

每個關鍵格的欄位意義：
- p：捲動進度 0–1
- f：影片第幾格
- vh：影片高度 ÷ 視窗高度；vw：影片寬度 ÷ 視窗寬度（兩者擇一）
- rx：影片右緣位置 ÷ 視窗寬；cx：影片中心 ÷ 視窗寬（兩者擇一）
- cy：影片中心 ÷ 視窗高
- cut：裁切開口的左緣落在「影片畫面」的哪個比例，用來遮掉影片內建的文字
- apR：開口右緣 ÷ 視窗寬，未指定時預設跟著影片右緣
- full：直式專用，開口永遠滿版

橫式 KEYS_LAND：

    { p:0.000, f:  0, vh:1.280, rx:0.965, cy:0.494, cut:0.500 }
    { p:0.190, f: 73, vh:1.280, rx:0.965, cy:0.494, cut:0.500 }
    { p:0.310, f:122, vh:1.090, rx:0.972, cy:0.492, cut:0.360 }
    { p:0.400, f:155, vh:1.060, cx:0.660, cy:0.496, cut:0.200, apR:1 }
    { p:0.470, f:181, vh:1.040, cx:0.545, cy:0.499, cut:0.083, apR:1 }
    { p:0.545, f:208, vh:1.028, cx:0.500, cy:0.500, cut:0.000, apR:1 }
    { p:0.660, f:250, vh:1.012, cx:0.500, cy:0.500, cut:0.000, apR:1 }
    { p:1.000, f:361, vh:1.000, cx:0.500, cy:0.500, cut:0.000, apR:1 }

直式 KEYS_PORT（開口滿版，改用「把畫面往左推」切掉內建文字，vw = 1/(1-cut)）：

    { p:0.000, f:  0, vw:2.0000, cy:0.470, cut:0.500, full:1 }
    { p:0.190, f: 73, vw:2.0000, cy:0.470, cut:0.500, full:1 }
    { p:0.310, f:122, vw:1.5625, cy:0.480, cut:0.360, full:1 }
    { p:0.400, f:155, vw:1.2500, cy:0.470, cut:0.200, full:1 }
    { p:0.470, f:181, vw:1.0900, cy:0.440, cut:0.083, full:1 }
    { p:0.545, f:208, vw:1.0000, cy:0.400, cut:0.000, full:1 }
    { p:0.660, f:250, vw:0.9850, cy:0.395, cut:0.000, full:1 }
    { p:1.000, f:361, vw:0.9800, cy:0.390, cut:0.000, full:1 }

換算成像素的規則（視窗尺寸改變時重算）：
- W = 視窗寬，H = .pin 的 clientHeight
- PAD = min(74, max(20, W*0.044))，對應 CSS 的 --pad
- portrait = (W/H < 1.05) || W < 760，成立時在 body 加 class portrait，並改用 KEYS_PORT
- 若有 vw：fw = vw*W; fh = fw/ASPECT；否則 fh = vh*H; fw = fh*ASPECT
- 位置：full 時 fx = -cut*fw；有 rx 時 fx = rx*W - fw；否則 fx = cx*W - fw/2；fy = cy*H - fh/2
- 開口左緣 ax0 = full ? 0 : max(0, fx + cut*fw)
- 開口右緣 ax1 = full ? W : min(W, apR!=null ? apR*W : fx+fw)

## 四、每幀要做的事

在 requestAnimationFrame 迴圈中，依 progress p：

1. 幾何：在分鏡表相鄰兩格間內插，緩動用 easeInOutCubic（t<.5 ? 4t³ : 1-(-2t+2)³/2）。
   - #film 套 transform（如上）
   - #stage 套 clip-path: inset(0px {W-ax1}px 0px {ax0}px)
2. CSS 變數：
   - --gate = ax0 px（開口分隔線位置）
   - --gate-o = clamp(cut/0.34, 0, 1)（分隔線透明度係數）
   - --copyw = clamp(edge - PAD*1.75, 236, cap) px，其中 edge = max(ax0, portrait?0:fx)、cap = min(980, max(760, W*0.34))。作用是影片開得越大，文字欄自動變窄。
   - --tr-right = p > 0.365 ? PAD : W - ax0 + 22（右上章節標記的位置；切換點刻意壓在 HUD 熄燈區間內，所以看不到它跳位）
   - 直式時 --scrim-h = p<=0.45 ? 74% : p>=0.74 ? 40% : 在兩者間用 easeInOutCubic 內插
3. 影片格：frame 用線性內插（不套緩動，捲動手感才會 1:1），want = clamp(frame/FPS, 0, DURATION - 1/FPS)
4. 色調：見下方 TONE / HUDO 表，同樣用 easeInOutCubic 內插，寫入 --bg / --ink / --hud
5. 章節：見下方 CHAPTERS
6. HUD：進度條寬度 = p*100%，百分比文字補零成兩位數
7. 捲動提示：p < 0.062 時顯示

效能要求：每幀都會算，但多數值沒變。必須用一個 memo 物件比對舊值，只有真的改變才寫回 DOM，避免每幀觸發整棵樹的樣式重算。

## 五、色調表

TONE（頁面背景與文字色，取樣自影片本身）：

    p:0.000  bg:#E7E2DD  ink:#14120F
    p:0.290  bg:#E0DAD4  ink:#14120F
    p:0.335  bg:#C6B0A4  ink:#14120F
    p:0.352  bg:#7A5C4C  ink:#5A5450
    p:0.378  bg:#2A1F1A  ink:#EFEBE7
    p:0.440  bg:#16110F  ink:#EFEBE7
    p:0.510  bg:#060607  ink:#F2F0EE
    p:0.600  bg:#020202  ink:#F2F0EE
    p:1.000  bg:#000000  ink:#F2F0EE

HUDO（HUD 透明度；明暗翻轉瞬間對比會塌掉，所以先熄燈再亮起）：

    p:0.000 o:1 / p:0.338 o:1 / p:0.352 o:.06 / p:0.374 o:.06 / p:0.396 o:1 / p:1.000 o:1

## 六、章節時序

CHAPTERS（in = 淡入區間，out = 淡出區間，皆為 progress）：

    { in:[0.000,0.000], out:[0.130,0.170], num:'00', name:'Identity'    }
    { in:[0.195,0.240], out:[0.300,0.332], num:'01', name:'What I Do'   }
    { in:[0.495,0.540], out:[0.620,0.660], num:'02', name:'How It Runs' }
    { in:[0.690,0.735], out:[1.100,1.100], num:'03', name:'The Stack'   }

判定：p >= in[0] && p < out[1] 時該章節加上 class is-in；p >= in[0] && p < out[0] 時它是「當前章節」，右上角 HUD 顯示其 num 與 name。

這裡有個坑：章節出場動畫的 CSS 選擇器必須寫成 #copy .ch > * 提高權重，否則會被 .lede、.sub 之類的單一 class 規則蓋掉，造成「非當前章節的文字仍然顯示」的鬼影。元素本身的淡化一律用 color-mix()，不要用 opacity。子元素進場用 --i 變數做階梯延遲（calc(var(--i) * 70ms)），退場時延遲必須歸零（transition-delay:0s; transition-duration:.4s），否則上一章的字會黏在畫面上。

## 七、文字內容（完全照抄）

章節 00 — Identity（data-ch="0"，這章底部要留空間給捲動提示：padding-top:44px; padding-bottom:124px）
- eyebrow：2026 — Portfolio
- h1（class big）：Data 換行 Engineer
- sub：資料工程師 / Data Engineer
- lede（三行，用 <br> 斷行）：
  專注於資料整合、資料倉儲與 ETL 自動化。
  把會員、訂單、流量與外部資料，
  變成能被決策直接使用的數據資產。
- meta 區塊：標籤 Featured Focus ／值 Data Platform // Automation（// 用橘色 class slash）／小字 Pipeline · Warehouse · Analytics

章節 01 — What I Do
- eyebrow：01 — What I Do
- h2（class mid）：讓資料， 換行 準時抵達。
- 三列條列（編號／標題／說明）：
  - 01 資料整合 Data Integration — 串接會員、訂單、流量與第三方來源，統一口徑。
  - 02 ETL / ELT 自動化 — 排程、重試、告警，讓管線自己跑完整夜。
  - 03 資料倉儲 Warehouse / OLAP — 分層建模，讓分析與報表能直接取用。

章節 02 — How It Runs
- eyebrow：02 — How It Runs
- h2：精密， 換行 來自可被拆解。
- lede：每一層都能單獨檢查、單獨重跑、單獨負責。 換行 所以出錯時，找得到；要擴充時，接得上。
- 標籤列：Pipeline / Warehouse / Analytics

章節 03 — The Stack
- eyebrow：03 — The Stack
- h2：一只錶， 換行 就是一座 + 橘色的 資料平台 + 。
- 六層索引（編號／英文／中文）：
  - 01 Sources — 會員 · 訂單 · 流量 · 外部 API
  - 02 Ingestion — Batch / Stream · ETL / ELT
  - 03 Processing — 清洗 · 轉換 · 加值
  - 04 Warehouse — 結構化儲存 · 分層建模
  - 05 Serving — API / BI / 報表 · 指標
  - 06 Governance — 品質 · 權限 · 稽核
- 這六層要逐層點亮：進度 0.700 → 0.960 之間依序，已過的加 class on（透明度 .42），當前的加 class now（透明度 1，編號與英文轉為橘色）。

HUD
- 左上：JASON HAN
- 右上：當前章節編號 + 名稱
- 左下：Progress + 一條進度線 + 百分比
- 捲動提示（p<0.062 時顯示，與文字欄同寬）：一條有橘色掃光來回的細線；下方一列 往下滑 + Scroll to explore + 一個圓形指示器，圈內是向下箭頭 SVG，外圈有橘色脈衝擴散，箭頭上下浮動

結尾區塊（.outro，深色 #050505，獨立於捲動舞台之外，min-height:100svh）
- eyebrow：04 — Contact
- h2：讓數據，成為 + 橘色的 決策的預設值 + 。
- 四欄卡片：
  - Name — JASON HAN 換行 資料工程師 · Data Engineer
  - Email — hanjason0926@gmail.com（mailto 連結，hover 變橘色）
  - Focus — Data Platform // Automation
  - Stack — Pipeline · Warehouse · Analytics
- 頁尾兩端對齊：左 © 2026 — Portfolio，右 Scroll-driven film · Lenis

## 八、視覺設定

CSS 變數：--bg:#E7E2DD、--ink:#14120F、--accent:#E4611F、--pad:clamp(20px, 4.4vw, 74px)。

字型：
- 英文 Inter（字重 300–900）
- 中文 Noto Sans TC（字重 300–700）
- 等寬（HUD、eyebrow、編號用）：ui-monospace, "SF Mono", "Cascadia Mono", "Segoe UI Mono", Consolas, monospace
- 從 Google Fonts 載入 Inter 與 Noto Sans TC，加 preconnect

排版特徵：
- eyebrow、HUD、標籤：等寬字、9–10px、字距 0.26–0.36em、全大寫，eyebrow 後面接一條短橫線
- 大標題 h1.big：字重 900、font-size:clamp(2.6rem, 18.6cqw, 9.6rem)、line-height:.85、letter-spacing:-.038em、全大寫
- 中標題 h2.mid：字重 700、clamp(1.35rem, 6.8cqw, 3.1rem)、line-height:1.24
- 內文 .lede：字重 300、line-height:2.05、max-width:34em
- 章節容器 .ch 要設 container-type:inline-size，上面的 cqw 單位才有作用
- 選取顏色 ::selection 用 accent 橘配白字
- favicon 用內嵌 data URI SVG：深色方塊 #090807 上一個橘色 #E4611F 空心圓環

質感層：
- 顆粒層 .grain：inset:-40px; z-index:6; opacity:.05; mix-blend-mode:overlay，背景是內嵌 data URI SVG 的 feTurbulence fractalNoise（baseFrequency .85、numOctaves 3）
- 開口分隔線 .gate：位於 left:var(--gate) 的 1px 直線，透明度 calc(var(--gate-o) * .14)

## 九、直式版差異（body.portrait）

- 文字改成壓在影片下方：.ch 改 left:0; right:0; width:auto; justify-content:flex-end
- 需要一層獨立的漸層壓暗層 .scrim（z-index:4，高度吃 --scrim-h，由下往上從 --bg 漸變到透明，中間用 color-mix 做 96%/80%/44% 三個過渡點）。這層一定要獨立成一個元素，放在 .ch 上會四個章節疊四次，整頁糊掉。
- 隱藏 .gate；條列的說明文字 .row .d 隱藏；六層索引改成兩欄格線且隱藏中文說明
- 捲動提示改為左右滿版、隱藏英文副標

## 十、載入流程

1. 開場有一個 loader（#boot，滿版蓋住，z-index:90）：Data Engineer 字樣 + 一條進度細線 + 兩位數百分比。body 初始帶 class is-booting（overflow:hidden）。
2. 依裝置挑檔案：portrait || navigator.connection?.saveData 為真時用 scroll-sm.mp4，否則用 scroll.mp4。
3. 輪詢緩衝進度（每 120ms，最多 300 次）：vid.buffered.end(最後一段) / DURATION。當緩衝 > 0.35 或 readyState >= 3 就放行；逾時也強制放行。
4. 放行時：進度拉到 100%、延遲 260ms 後 loader 淡出、移除 is-booting、lenis.start()。
5. 影片 error 事件 → #stage 加 class no-video，改用 poster.jpg 當靜態底圖（background-image），並隱藏 video 元素。
6. Lenis 設定：lerp: 0.085、wheelMultiplier: 1、touchMultiplier: 1.6、smoothWheel: true、autoRaf: false（自己在主迴圈呼叫 lenis.raf(time)）。啟動前先 lenis.stop()。
7. 重新整理時 history.scrollRestoration = 'manual' 並 window.scrollTo(0,0)，避免中途載入造成分鏡錯位。
8. resize 事件用 120ms debounce，重算幾何 + lenis.resize() + 重新套用當前 progress。

## 十一、seek 節流

影片 seek 不能每幀都發，否則會塞車：

- 用 busy 旗標，正在 seek 時直接 return
- 目標與上次 seek 差距小於半格（0.5/FPS）就跳過
- 優先用 vid.fastSeek(t)，不支援才退回 vid.currentTime = t，整段包 try/catch
- 監聽 seeked 事件解除 busy 並立刻再跑一次 pump
- 另外掛一個 220ms 的 setTimeout 保險，避免 seeked 沒觸發時永久卡住

## 十二、無障礙

- 尊重 prefers-reduced-motion:reduce：Lenis 的 lerp 設為 1、關閉 smoothWheel、章節轉場縮到 0.01ms、捲動提示的掃光與浮動動畫停止
- 影片加 aria-hidden="true"（純裝飾）
- body 設 overscroll-behavior-y:none; overflow-x:hidden
- .pin 高度用 100svh，避免手機網址列縮放時抖動
- <meta name="viewport"> 加 viewport-fit=cover
- 語言 zh-Hant，標題 DATA ENGINEER — Portfolio

請直接輸出完整的 index.html，不要省略任何段落，不要用「其餘同上」之類的縮寫。
```

### 使用備註

- **影片網址是硬性依賴**：如果哪天 repo 改名或 GitHub Pages 關閉，網址會失效，產出的頁面就只剩 `poster.jpg` 降級底圖。
- **分鏡表數值是量身訂做的**：`cut:0.500` 是量出來的安全值 —— 影片內建文字在第 0 格最遠延伸到畫面寬的 48.6%，而人物頭髮最左在 51.4%（第 25 格），切在 50% 兩邊都不傷。換一支影片就必須重新量。
- **Lenis 版本已釘死在 1.1.13**：`autoRaf` 選項是 1.1.x 才有的，不要改成 `@latest`。
