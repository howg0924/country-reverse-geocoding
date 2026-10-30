# country-reverse-geocoding（fork）

離線的經緯度→國家（以及美國州）反查：內建 GeoJSON 資料，逐一做 point-in-polygon。fork 自 `teleportd/country-reverse-geocoding`（原作者 Guillaume Flandre），分支 `master`。

## ⚠️ 這是公開 repo
任何人都看得到本 repo 的檔案、commit 訊息與歷史。寫進這裡的一切（程式、文件、commit 訊息、PR／issue 文字）都要能公開：
- 只寫套件本身的功能、與上游的差異、套件自己遇到的問題與修法。
- **不要寫**：哪些專案、App 或服務在使用本套件（含其名稱、目錄、檔名、函式）、使用端釘住的版本 hash、測試機或環境的實測資料、票號與內部的問題編號、內部文件或工作區的路徑、帳號與主機名稱。commit 訊息也一樣，不要加內部票號前綴。
- 使用端專屬的細節（怎麼升 pin、在哪驗證、哪個畫面用到、待確認事項）寫在**使用端專案自己的文件**，不要寫在這裡。
- 不確定某段資訊適不適合公開，就不要寫，先問使用者。push 前再看一遍即將推出的 diff 與 commit 訊息。

## 與上游的差異
- 只有 2 個 fork 的 commit，都只改 `lib/country_reverse_geocoding.js` 的資料：
  - `3be1f31`（2020-02-07）重排 `my.country_data` 順序（`TWN`、`CHN`、`MYS`、`JPN`、`NGA`、`USA`、`SGP`、`AUS`、`THA` 移到最前面）以加速搜尋。
  - `5022d5b`（2020-04-09）把 `TWN` 的多邊形換成一個較大的自繪多邊形（頂點為 7 位小數，與其餘上游資料不同），修正「部分台灣地區不被判成台灣」。
- 其餘（含 `9a08fd8` 加入 `SGP`）是上游的 commit。上游 `README.md` 為準，但它只提到 `get_country`，實際程式還有 `get_us_state`。

## 技術棧與入口
- CommonJS，單檔 `lib/country_reverse_geocoding.js`（`main`），依賴 `fwk 1.0.x`（僅用來掛 `fwk.method`）。用法：`require('country-reverse-geocoding').country_reverse_geocoding()` 取得 `{get_country(lat, lng), get_us_state(lat, lng)}`。
- 檔案約 350 KB，其中 `my.us_data`（52 筆）與 `my.country_data`（180 筆）的每一筆 GeoJSON Feature 都是**單獨一行、最長約 2.6 萬字元**：不要整檔 `Read`／`cat`。列出資料順序：`tr -d '\r' < lib/country_reverse_geocoding.js | grep -o '^{"type":"Feature","id":"[A-Za-z-]*"'`；看程式邏輯：`... | cut -c1-140 | awk '!/^\{"type":"Feature"/'`。

## 行為（改資料前必知）
- 參數順序是 `(lat, lng)`，但 GeoJSON 座標是 `[lng, lat]`（程式內 `point = [lng, lat]`）。
- 回傳 `{code, name}`（`code` 是資料裡的 `id`，三碼國碼）、找不到回 `null`、參數不是 number 時**回傳**（不是 throw）一個 `Error`；呼叫端要自己處理這個回傳值。
- 演算法是依陣列順序做第一個命中的線性掃描，且 MultiPolygon／Polygon 都只檢查外環 `coordinates[0]`（不處理洞）。因此**陣列順序就是重疊或粗糙多邊形的優先順序**，順序不可隨意調整。
- 呼叫端可能把 `id` 當字串比較（例如 `'TWN'`／`'USA'`），不要改動既有 `id`。資料內 `CYP`、`SOM` 各有兩筆同 id 的資料。

## 常用指令與測試
- 沒有 npm scripts、沒有 devDependencies；測試是 `spec/country_reverse_geocoding.spec.js`（jasmine-node，上游 README 的做法：全域安裝 `jasmine-node` 後 `jasmine-node spec/`）。本機沒有 `node_modules`。
- 2026-10 以離線 shim（stub `fwk`、自製 `describe`／`it`／`expect`）實跑該 spec：25 項中 24 項通過，失敗的一項是新加坡座標被判成馬來西亞（資料順序讓 `MYS` 排在 `SGP` 前，見地雷）。
- 回歸用點位（HEAD 應為 `TWN`）：台北、台中、花蓮、高雄、澎湖、綠島、蘭嶼、屏東南端；廈門應為 `CHN`。金門、馬祖、新加坡目前的結果見地雷；舊版資料（`3be1f31`）對高雄、澎湖等地回傳 `null`。

## 發佈與引用方式
- 使用端以 git URL 引用，實際版本由使用端 `yarn.lock` 的 `resolved ...#<hash>` 決定。
- 流程：改 `lib/` → commit → push 到 `origin/master` → 使用端更新 `yarn.lock` 的 pin。agent 改完不要自行 push，除非使用者明確要求。

## 必守規則
- 動資料或調整順序前，先用上面的指令確認現有順序，改完用實測點位（台灣各地、新加坡、法國等）驗證：順序決定重疊多邊形的結果。
- 新增或修正國家時，同時在 `spec/country_reverse_geocoding.spec.js` 加一個點位斷言（`9a08fd8` 加 `SGP` 時就是這樣做的）。
- 不要「整理」資料行（重新排序、重新格式化、拆行）：會讓 diff 失真，且順序有語意。

## 地雷
- **重疊多邊形的結果由資料順序與 `TWN` 的範圍決定**：目前已知兩個案例：`MYS` 排在 `SGP` 前，新加坡座標被判成馬來西亞；`TWN` 多邊形未涵蓋金門與馬祖，金門判成 `CHN`、馬祖判成 `null`。改順序或 `TWN` 多邊形前先確認影響；已部署的版本都帶著現狀，下游可能已依賴。
- `package.json` 的 `repository`、`homepage` 仍指向上游 `teleportd`，`version` 是 0.2.1（fork 的 commit 沒有 bump 過）；辨識版本只能靠 commit hash。
- 模組載入時就把整份資料建進記憶體；查不到點（`null`）會掃完整個陣列，是最慢的路徑，每次最壞掃描 180 國。只檢查外環、沒有邊界框預檢，所以小國與飛地的判定不準。

## 延伸文件（需要時才讀）
- `README.md` — 上游說明（API 與資料來源 `johan/world.geo.json`），不要改寫
