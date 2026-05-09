# 拆分與重構建議

目前 [index.html](index.html) 是單檔 ~880 行,維持單檔的優勢是「拖進瀏覽器即玩、零部署、零依賴」。
本文件記錄痛點出現時的處理方向,**不是現在的待辦清單**。

---

## 何時動手

達成任一條件再開始拆,別因為「應該拆」就動工:

- JS 超過 1500 行,或單一 class 超過 400 行
- 想寫單元測試(class 需要 export 才好測)
- 想引入 TypeScript / Vite / HMR
- 多人共同維護,git conflict 頻繁
- 想做關卡資料化(關卡設定從程式碼抽到 JSON)

---

## 推薦拆分順序

### 階段 1:CSS / JS 抽出(最小痛苦,不破壞部署方式)

維持「拖檔即玩」,只把長度切開:

```
index.html      # 結構 + <link> + <script src>
style.css       # 全部 CSS
game.js         # 全部 JS,順序不變
```

- 不用 ES modules、不需要 server
- 缺點:`game.js` 仍是長單檔,治標不治本
- 適合:純粹覺得 index.html 太長,想分段編輯

### 階段 2:ES Modules 依 class 拆檔

此階段起需要本機 server(`python3 -m http.server` 或 Vite dev)——`file://` 不允許 import。

```
index.html
style.css
src/
├── main.js          # 啟動入口,只 new Game()
├── config.js        # Config, SPECIAL, GAME_STATE 常數
├── utils.js         # Utils
├── audio.js         # AudioEngine
├── particles.js     # ParticleEngine
├── board.js         # Board, MatchFinder
├── renderer.js      # Renderer
└── game.js          # Game (controller)
```

依賴方向(由下往上 import):

```
config, utils                    # 葉節點
  ↓
audio, particles, board, renderer
  ↓
game
  ↓
main
```

### 階段 3:打包工具與型別(可選)

- **Vite**:HMR、自動 minify、import map
- **TypeScript**:Cell / Run / SpecialPlan 等資料結構受惠最大
- **Vitest**:單元測試 MatchFinder、Board(純資料邏輯,最好測)

---

## CSS 拆法(若 CSS 也想模組化)

按功能塊分檔,用 `@import` 串起:

```
styles/
├── base.css     # 變數、body、背景、動畫 keyframes
├── hud.css      # top-bar、HUD、modal、overlay
├── candy.css    # candy 顏色、特殊糖樣式
└── fx.css       # 粒子、shake、flash、shuffle 等特效
```

---

## 重構機會清單(拆檔時順便處理)

### 高價值

- **AudioEngine 改用 SFX 表**:目前每個音效一個方法,改成 `audio.play('match' | 'combo' | ...)` + 設定物件,調音時集中改一處
  ```js
  const SFX = {
      match: { type: 'glide', f: [720, 1080], dur: 0.14, ... },
      combo: { type: 'arp', baseFreq: 600, ... },
  };
  ```
- **Input 邏輯獨立**:[`_bindInputs`](index.html) 混雜 swipe / tap / double-tap / pointer 狀態,抽 `InputHandler` class 後測試與調整都更容易
- **關卡資料化**:目前 `targetScore = 500 + (level - 1) * 100` 寫死在 `claimReward`。抽成 `levels.json`(或 `levels.js` 陣列)後,新增關卡不用改邏輯

### 中價值

- **Renderer dirty tracking**:目前每次 `render()` 重繪 64 cell。8x8 不痛,板子變大或加入更多裝飾元素時可改為只更新變動 cell
- **State machine 集中化**:`if (this.state !== GAME_STATE.PLAYING) return` 散落多處,集中為 `_canAct()` 或 transition 函式
- **Storage 抽象**:`localStorage.getItem(...)` / `setItem(...)` 多處重複,可抽 `storage.js` 集中 key prefix 與版本管理

### 低價值(現在不要做)

- **引入 React / Vue / Svelte**:這個遊戲是純 imperative DOM 操作,框架反而拖累。除非想做選單/設定畫面才考慮
- **Service Worker / PWA**:無離線需求,單檔本來就會被瀏覽器 cache
- **WebGL 升級粒子系統**:目前 Canvas 2D 跑得動,粒子數沒到瓶頸

---

## 反模式(避免做)

- **過早抽象**:出現「以後可能會...」的念頭就停下,真的需要時再抽
- **每個 class 拆一檔但互相 import 形成毛球**:Board + MatchFinder 同檔即可(它們本來就強耦合)
- **不必要的依賴**:目前零依賴。每加一個 npm package 都該問「真的非它不可嗎?」

---

## 拆分前的最後檢查

開始拆之前,確認:

- [ ] 已有可重現的測試流程(至少能手動跑完三個模式)
- [ ] git working tree 乾淨,有獨立 branch
- [ ] 知道部署環境是否容許 server(若仍只能 `file://`,卡在階段 1)
- [ ] 拆完之後新增功能會明顯變快(否則拆了沒意義)
