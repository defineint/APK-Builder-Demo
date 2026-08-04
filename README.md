# 零門檻 APK 自動建置系統 (Zero-Threshold APK Builder)

## 1. 專案定位 (Project Overview)

這是一款對話式 Android APK 自動建置工具。使用者只需透過自然語言描述應用程式需求，系統便會自動進行需求分析、生成藍圖、撰寫 Kotlin 程式碼、執行自動修復與測試，並最終編譯打包成可下載的 APK 檔案。

---

## 2. 系統總體架構 (System Architecture)

<img width="1303" height="764" alt="專題架構圖" src="https://github.com/user-attachments/assets/7a155b40-9434-422b-bb6c-fe9ac08ed4e5" />

---

## 3. 核心亮點：LLM 幻覺防護與自動化修復引擎

為了解決大型語言模型在生成程式碼時常見的錯誤與不穩定性，本系統後端實作了多層次的防護與自動修正機制：

* **階段一：藍圖化與架構解耦**
  * 由 Orchestrator Service（編排者）將自然語言轉化為結構化 JSON 藍圖，將需求解析與程式碼生成拆分處理。
  * 透過 prompt 規範 LLM 盡可能揣測使用者意圖，若使用者語意不清或是與 APP 實作不相關，則引導使用者給予正確資訊
* **階段二：靜態語法清洗**
  * 在解析與修正模組 (QA & Parser) 中，實作正規表達式 (Regex) 清洗，並結合 `ktlint` 進行 Kotlin 代碼的格式化與語法自動修復。
* **階段三：自動修復循環 (Auto-Repair Loop)**
  * 結合 APK Builder 建置引擎，執行自動修復循環，透過正規表達示 (Regex) 擷取錯誤訊息，並且將該錯誤回報 LLM 重新生成邏輯程式碼
* **階段四：TDD 邏輯測試**
  * 除了生成邏輯程式外，系統也會同步產生 JUnit 測試代碼 (TDD 流程)，在確認邏輯程式碼通過測試後，才會著手建立系統的 UI 介面。

---

## 4. 模組分工與技術棧 (Core Modules & Tech Stack)

本專案採用前後端分離架構，結合 LLM 生成技術與 Android 原生建置工具鏈。

### 前端應用 (Frontend)

* **核心框架** ：Flutter (3.10+) / Dart。採用 StatefulWidget 與 Service-based 邏輯分離。
* **狀態與非同步處理** ：實作長效輪詢 (Long-polling) 與 `onUpdate` 回呼機制，即時更新 `ChatHistory` 列表，並動態渲染 APK 下載按鈕，避免阻塞 UI 線程。
* **第三方整合**：
  * `supabase_flutter`：處理使用者身分驗證 (OAuth2 / Email) 與資料庫互動。
  * `app_links` & `windows_single_instance`：支援 Deep Link 攔截與 Windows 單例應用特性。
  * `http` & `url_launcher`：負責 RESTful API 通訊與 APK 下載連結跳轉。

### 後端引擎 (Backend)

* **核心框架**：Python 3.10+ / FastAPI。
* **非同步與併發處理**：透過 `asyncio` 實作非同步任務引擎，支援多使用者並行請求 (Concurrency)。建置任務配置於獨立 Task Queue，透過 UUID 追蹤進度，並經由 SSE/WebSocket 即時回傳狀態 (`[DONE]`, `[ERROR]`)。
* **LLM 整合**：使用 `Gemma 4 26B A4B` 本地模型處理自然語言轉藍圖與程式碼生成。支援「純編譯模式」，可跳過 LLM 階段直接從資料庫載入既有代碼編譯。

### 資料庫與後端即時服務 (BaaS)

* **平台**：Supabase (PostgreSQL + Supabase Storage)。
* **資料庫架構**：
  <img width="1295" height="440" alt="image" src="https://github.com/user-attachments/assets/7665230c-0bc4-47cb-aee9-f382b1920833" />
* **功能應用**：
  * 統一處理身分驗證 (Auth)。
  * 負責儲存對話歷史、使用者資訊與生成的原始碼等。
  * **雲端檔案託管**：建置完成的 APK 會自動上傳至 **Supabase Storage** (`apk-outputs` Bucket)，並產生具時效性的安全下載連結供前端存取。

---

## 5. 資源管理與安全防護 (Resource Management & Security)

為確保伺服器穩定性與防範資源溢滿，本系統實作了以下空間最佳化與安全機制：

* **全自動資源回收機制 (Garbage Collector)**：
  * **雲端空間釋放**：定期掃描 Supabase，自動呼叫 Storage API 刪除已過期之 APK 下載連結或已刪除專案的雲端檔案。
  * **本地工作區清理**：監控 Gradle 工作目錄 (`WORKSPACES_DIR`)，自動移除閒置超過 12 小時的專案編譯目錄；並定期清理 `STATIC_DIR` 中存放超過 24 小時的 `.kt` 暫存原始碼。
* **編譯效能最佳化**：
  * 針對 Android SDK & Gradle 配置**共用 Gradle Cache (GRADLE_USER_HOME)**，避免重複下載依賴檔案，大幅提升編譯速度並節省 Docker 伺服器磁碟空間。
* **環境變數隔離**：
  * 嚴格分離前後端權限，後端透過 `python-dotenv` 載入最高權限之 `SUPABASE_SERVICE_ROLE_KEY` 進行安全管控。

---

## 6. 專案狀態與未來展望 (Status & Future Work)

* **目前狀態**：由於本專題尚未打算公開，原始碼暫時設為私有。如需技術交流或查看 Demo，歡迎聯繫：`paulag3seconds@gmail.com`
* **未來展望 (Future Work)**：
  * 提供預覽功能，讓使用者透過視覺回饋精準引導 UI 設計。
  * 支援圖片上傳與視覺審查，讓 AI 精準對齊使用者期待的 UI 藍圖。
  * 透過訓練模型的方式，從「單一巨型模型」走向「輕量化多模型分工」，解放算力並倍增產出效率。
  
