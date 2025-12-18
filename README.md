# 軟體工程期末報告  11224204林柏廷,11224210賴宣妤


## 題目: 停車場管理系統

### 需求說明書
#### 1. 引言 (Introduction)
##### 1.1 專案目的
本系統旨在開發一套結合物聯網 (IoT) 硬體與軟體介面的智慧停車場管理解決方案。除了基本的自動化進出與計費功能外，特別著重於特殊車位（如無障礙車位）的智慧監控與管理，解決身障車位遭非授權車輛佔用的問題，提升停車場營運效率與公平性。

##### 1.2 系統範疇
系統包含三個主要子系統：

地端硬體控制層： 使用微控制器 (ESP32/Arduino) 連接超音波/紅外線感測器、相機、閘門馬達與警示燈。

後端伺服器層： 負責商業邏輯、MySQL 資料庫存取、影像處理 (OpenCV/AI)。

前端應用層： 管理員網頁 (Web) 與使用者 APP (App Inventor/Mobile)。

#### 2. 使用者功能需求 (User Functional Requirements)

![03](https://github.com/11224204lbt/parking-system/blob/main/UML%20%E7%94%A8%E4%BE%8B%E5%9C%96.png)

#### 圖 0：停車場管理系統用例圖

功能需求依據使用者角色分為三大類：

##### 2.1 一般車主 / 駕駛 (End User)
車牌辨識進場： 車輛駛入入口時，系統需自動觸發影像辨識，無需取票即可入場（記錄入場時間、車牌號碼）。

車位導引與狀態查詢： 使用者可透過 APP 或入口顯示屏，查看剩餘車位數量。

自助繳費： 支援輸入車牌號碼查詢停車費用，並進行模擬支付（專題通常使用模擬金流）。

無感出場： 繳費完成後，出場時系統辨識車牌自動開啟閘門。
![03](https://github.com/11224204lbt/parking-system/blob/main/4.jpg)
#### 圖1：一般車主停車與繳費情境示意圖 

##### 2.2 無障礙車位使用者 (Accessible Parking User) - 專題核心亮點
身分驗證： 當車輛停入無障礙車位時，系統需結合車牌辨識或 RFID 識別證，確認該車輛是否具備身障停車資格。

違規佔用警示： 若偵測到非授權車輛佔用無障礙車位（例如：感測器偵測到有車，但車牌不在白名單內），現場硬體需發出聲光警示（如蜂鳴器響、紅燈閃爍）。

##### 2.3 系統管理員 (Administrator)
儀表板監控 (Dashboard)： 顯示即時車位狀態（一般車位 vs 無障礙車位佔用率）、今日營收、異常事件列表。

車位狀態管理： 可遠端手動重置車位狀態（針對感測器誤判情況）。

報表查詢： 查詢歷史進出紀錄、違規佔用紀錄（附照片證據）、營收報表。

費率設定： 可調整計費標準（如每小時費率、免費時段）。

#### 3. 性能需求 (Performance Requirements)
此部分定義系統的「速度」與「穩定性」，是資工專題評分的重要指標。

##### 3.1 時間特性 (Time Constraints)
車牌辨識速度： 從車輛觸發感測器到完成辨識並送出開閘訊號，延遲時間需小於 2 秒。

資料庫回應時間： 前端查詢剩餘車位或計費請求，資料庫 Query 回應需在 500ms (0.5秒) 內完成。

硬體感測延遲： 無障礙車位感測器 (如超音波) 狀態改變後，後端資料庫需在 1 秒內 更新狀態。

##### 3.2 準確性 (Accuracy)
車位狀態準確度： 硬體感測器對於「有車/無車」的判斷準確度需達 95% 以上（需處理誤判，如路人經過）。

車牌辨識率： 在標準光源環境下，OpenCV/AI 模型的辨識準確率需達 90% 以上。

##### 3.3 可靠性 (Reliability)
例外處理： 若網路中斷，地端硬體 (ESP32) 需具備離線模式，可暫存進出紀錄，待網路恢復後自動上傳 MySQL。

#### 4. 介面要求 (Interface Requirements)
##### 4.1 使用者介面 (GUI) - Web/App
視覺化車位圖 (Visual Map)：

管理員介面需繪製停車場平面圖。

使用顏色區分狀態：<span style="color:green">綠色(空位)</span>、<span style="color:red">紅色(已停)</span>、<span style="color:orange">橘色(違規佔用警示)</span>。

響應式設計 (RWD)： 管理介面需適配桌機與平板，方便管理員移動巡視。

操作回饋： 所有按鈕點擊後需有 Loading 動畫或 Toast 訊息提示（如：「修改成功」、「連線逾時」）。

##### 4.2 硬體介面 (Hardware Interface)
警示裝置： 無障礙車位旁需設置 LED 燈條與蜂鳴器。

合法停放： 亮綠燈/熄燈。

違規停放： 亮紅燈並發出警示音。

通訊協定：

ESP32 與後端伺服器之間使用 Wi-Fi (HTTP RESTful API 或 MQTT) 進行通訊。

資料格式統一使用 JSON。

##### 4.3 資料庫介面 (Database Interface)
使用 MySQL 作為關聯式資料庫。

需設計正規化 (Normalization) 至少達 3NF 的 Schema，包含 Parking_Lots, Records, Users, Violations 等資料表。

### 概要設計說明書
#### 1. 系統架構設計 (System Architecture Design)
本系統採用 B/S (Browser/Server) 與 Client/Server 混合架構，整體邏輯分層如下：

##### 1.1 邏輯架構圖 (Logical Architecture) - 三層式架構
系統分為 感知層 (Perception/Client)、業務邏輯層 (Business Logic) 與 資料層 (Data)。

感知層 (Client Tier & IoT Edge)：

硬體端： ESP32/Arduino + 超音波感測器 + 攝影機 + 閘門伺服馬達 + 警示燈/蜂鳴器。負責現場資料蒐集與物理控制。

軟體端： App Inventor (使用者端 APP)、Web 瀏覽器 (管理員後台)。負責人機互動。

業務邏輯層 (Application Tier)：

後端伺服器 (Backend Server)： 執行 Python (Flask/Django) 或 Node.js。負責接收硬體訊號、執行 OpenCV 影像辨識、邏輯判斷 (如：是否違規佔用)、API 回應。

資料層 (Data Tier)：

資料庫： MySQL。負責儲存使用者資料、停車紀錄、違規影像路徑與費率設定。

##### 1.2 實體部署架構 (Physical Architecture)
地端 (Local): 停車場入口閘門控制器、無障礙車位監測模組 (ESP32)。

雲端/伺服器端 (Server): 部署於實驗室伺服器或雲端 (如 AWS/GCP)，運行 Web Server 與 Database。

![03](https://github.com/11224204lbt/parking-system/blob/main/UML%20%E7%B3%BB%E7%B5%B1%E9%83%A8%E7%BD%B2%E5%9C%96.draw.io.png)

#### 圖 X：系統實體部署架構圖

#### 2. 系統模組劃分 (Module Decomposition)
為了方便分工開發，將系統劃分為以下四大核心模組：

##### 2.1 硬體控制模組 (Hardware Control Module)
功能： 直接控制電子元件。

子模組：

感測子系統： 讀取超音波距離數據，判斷車位是否有車。

執行子系統： 控制伺服馬達 (閘門開關)、控制 LED/蜂鳴器 (違規警示)。

通訊子系統： 透過 Wi-Fi (HTTP/MQTT) 將狀態傳送至後端。

![03](https://github.com/11224204lbt/parking-system/blob/main/UML%20%E7%A1%AC%E9%AB%94%E6%8E%A5%E7%B7%9A%E5%9C%96.draw.io.png)

#### 圖 3：地端硬體接線示意圖

##### 2.2 影像處理模組 (Image Processing Module)
功能： 處理進出場與車位監控的影像分析。

子模組：

車牌辨識 (LPR Core)： 接收影像 -> 灰階化/二值化 -> 車牌定位 -> 字元分割 -> OCR 辨識 -> 輸出車牌字串。

違規偵測 (Violation logic)： (針對無障礙車位) 當硬體感測到有車，但車牌辨識結果不在「身障白名單」內，標記為違規。

##### 2.3 後端管理模組 (Backend Management Module)
功能： 系統的大腦，處理所有邏輯運算。

子模組：

計費引擎： 計算 (出場時間 - 入場時間) * 費率。

車位狀態管理： 接收硬體訊號，更新資料庫中的 status (空/滿/違規)。

API 介面： 提供 RESTful API 供 App 和硬體呼叫。

##### 2.4 前端互動模組 (Frontend Interaction Module)
功能： 提供視覺化介面。

子模組：

管理員儀表板 (Web)： 顯示即時車位圖、營收報表、強制開閘按鈕。

使用者 APP (App Inventor)： 查詢剩餘車位、試算停車費。
![03](https://github.com/11224204lbt/parking-system/blob/main/3.jpg)
#### 圖2：使用者 App 操作業務流程圖

#### 3. 介面設計 (Interface Design)
##### 3.1 外部介面 (External Interface) - API 設計
定義硬體與軟體、前端與後端溝通的標準。採用 HTTP RESTful 風格，回傳格式為 JSON。

![03](https://github.com/11224204lbt/parking-system/blob/main/1.jpg)
#### 圖3

##### 3.2 內部介面 (Internal Interface) - 資料庫存取
後端程式透過 SQL Connector (如 Python 的 mysql-connector 或 SQLAlchemy) 與 MySQL 溝通。

主要資料表 (Entities):

Users (使用者/白名單)

ParkingSpots (車位資訊)

Records (進出紀錄)

Violations (違規紀錄)

##### 3.3 人機介面 (User Interface, UI) 設計規範
管理後台：

採用「左側選單、右側內容」的佈局。

無障礙車位監控頁面： 需將無障礙車位特別標示（例如使用輪椅圖示），若違規需閃爍紅色邊框。

使用者 APP：

首頁即顯示大字體的「剩餘車位數」，避免駕駛分心。

#### 4. 資料庫概要設計 (Data Design Overview)
(詳細的 Schema 會在詳細設計階段定義，此處先定義實體關係)

系統主要包含以下實體 (Entities)：

車位 (Spot): 屬性包含編號、類型 (一般/無障礙)、目前狀態。

進出紀錄 (Record): 屬性包含入場時間、出場時間、車牌、費用。

車輛 (Vehicle): 屬性包含車牌號碼、車主資訊、是否為身障車輛 (白名單)。

關鍵關聯 (Relationship):

一個 車位 可以有多筆 進出紀錄 (1:N)。

一台 車輛 可以有多筆 進出紀錄 (1:N)。

#### 5. 技術選型 (Technology Stack)
這是根據你目前的專題規劃所建議的技術清單：

前端 (Frontend): HTML/CSS/JS (管理後台), App Inventor (手機 App)。

後端 (Backend): Python (Flask) 或 Node.js (Express)。

資料庫 (Database): MySQL 8.0。

硬體 (Hardware): ESP32 (主控), HC-SR04 (超音波), Servo (伺服馬達), WebCam (影像輸入)。

演算法 (Algorithm): OpenCV (影像處理), Tesseract OCR (文字辨識)。

### 詳細設計說明書
#### 1. 資料庫詳細設計 (Database Physical Design)
本系統使用 MySQL 資料庫。以下是將概念轉化為實際 SQL 表格的結構設計。

##### 1.1 實體關聯圖 (ER Diagram)
這張圖定義了資料表 (Table) 的欄位 (Column)、主鍵 (PK) 與外鍵 (FK)。

![03](https://github.com/11224204lbt/parking-system/blob/main/%E8%BB%9F%E5%B7%A5ER%E5%9C%96.png)

##### 1.2 資料表規格說明
Users (白名單): 儲存身障人士資訊，用於判斷是否違規。

Records (流水帳): 當車輛進場時，exit_time 與 fee 預設為 NULL/0，直到出場更新。

Violations (違規): 專門記錄無障礙車位被非白名單車輛佔用的事件。

#### 2. API 介面詳細設計 (Interface Specification)
定義前端 (App/Web) 與地端硬體 (ESP32) 如何與後端伺服器交換資料。採用 JSON 格式。

##### 2.1 硬體回報 API (ESP32 -> Server)
用途： 當超音波感測器狀態改變時呼叫。

Endpoint: POST /api/hardware/update_status

Request Body (JSON):
    {
      "spot_id": "Disable_01",
      "distance": 30.5,        // 單位: cm
      "is_occupied": true      // true=有車, false=無車
    }
Logic: 後端收到 is_occupied: true 且該車位是無障礙車位時，觸發「違規檢查流程」。

##### 2.2 違規警示 API (Server -> ESP32)
用途： 後端判斷違規後，命令 ESP32 亮紅燈/鳴叫。 (註：通常由 ESP32 輪詢或透過 MQTT 訂閱 Topic，以下以回應模式為例)

Response (JSON):
    {
      "status": "success",
      "action_command": "ALERT_ON"  // 或 "ALERT_OFF"
    }
##### 2.3 車位查詢 API (App -> Server)
用途： 手機端查詢剩餘車位。

Endpoint: GET /api/spots/status

Response (JSON):
    {
      "total_spots": 50,
      "available_spots": 12,
      "spots": [
        {"id": "A01", "status": "empty"},
        {"id": "Disable_01", "status": "occupied"}
      ]
    }
#### 3. 模組演算法與邏輯流程 (Algorithms & Logic Flow)
這是程式實作的核心，我們針對兩個最關鍵的邏輯進行詳細描述。

##### 3.1 演算法一：感測器訊號濾波 (防誤判演算法)
問題： 超音波感測器可能會因為路人經過或訊號抖動，導致數值瞬間跳動，造成系統誤判有車。 解決方案： 實作「時間視窗濾波 (Time Window Filtering)」邏輯。

邏輯描述：

連續偵測： ESP32 每 500ms 讀取一次距離。

狀態計數器：

若 距離 < 閾值(50cm)，計數器 Count + 1。

若 距離 > 閾值，計數器歸零。

觸發判定：

當 Count 累積超過 5 次 (即持續 2.5 秒都有障礙物)，才正式判定為 Occupied (有車)。

避免路人經過瞬間觸發誤報。

##### 3.2 演算法二：無障礙車位違規判斷邏輯 (Core Logic)
問題： 如何判斷停在無障礙車位上的車是否違規？ 輸入： 車位 ID、當下拍攝的車牌影像。 輸出： 是否違規 (Boolean)、控制指令。

![03](https://github.com/11224204lbt/parking-system/blob/main/UML%20%E5%BE%AA%E5%BA%8F%E5%9C%96.drawio.png)

#### 圖4：無障礙車位違規偵測時序圖

#### 4. 模組封裝建議 (Implementation Suggestions)
為了讓程式碼整潔，建議在詳細設計階段就定義好 Python 的 Class 結構：

class HardwareController: 負責處理 MQTT/HTTP 訊號，不含業務邏輯。

class LPRService: 封裝 OpenCV 功能，輸入圖片，輸出字串。

class ParkingManager: 核心邏輯層，負責呼叫資料庫並決定「是否違規」。

![03](https://github.com/11224204lbt/parking-system/blob/main/UML%20%E9%A1%9E%E5%88%A5%E5%9C%96.drawio.png)

#### 圖 5：後端系統類別結構圖

### 測試計畫書
#### 1 測試目標與範圍 (Objectives & Scope)
##### 1.1 測試目標
本計畫旨在驗證「智慧停車場管理系統」之軟硬體整合穩定性與功能正確性。主要目標包含：

功能驗證： 確保車牌辨識進出、計費邏輯、無障礙車位違規偵測等核心功能運作正常。

效能評估： 驗證系統反應時間（如：辨識至開閘 < 2秒）是否符合需求。

可靠性測試： 確保在地端網路波動或硬體異常時，系統具備基本的錯誤處理能力。

##### 1.2 測試範圍
單元測試 (Unit Test)： 針對 ESP32 感測器讀數、伺服馬達控制、Python 資料庫存取函式進行獨立測試。

整合測試 (Integration Test)： 驗證 ESP32 與後端 Server 的 API 資料傳輸，以及 Server 與資料庫的連動。

系統測試 (System Test)： 模擬真實使用者情境，進行端對端 (End-to-End) 的完整流程測試。

#### 2 測試環境配置 (Test Environment)
為模擬真實運作場景，本專案建立了一套微縮模型進行測試。詳細軟硬體配置如下表所示：

![03](https://github.com/11224204lbt/parking-system/blob/main/2.jpg)

#### 圖6

#### 3 測試策略 (Test Strategy)
本專案採用 由下而上 (Bottom-Up) 的測試策略，先確保底層硬體與單一模組無誤，再進行整體流程驗證。

硬體單元測試： 使用 Arduino Serial Monitor 監控超音波數值與馬達角度，確保感測器無誤動作。

API 介面測試： 使用 Postman 發送模擬的 JSON 請求至後端 Server，檢查回應狀態碼 (200 OK) 與資料庫寫入結果。

全系統情境測試： 實際操作玩具車進行進出場與違規停車，觀察系統反應。

#### 4 測試案例 (Test Cases)
以下列出最具代表性的關鍵功能測試案例：

![03](https://github.com/11224204lbt/parking-system/blob/main/%E6%B8%AC%E8%A9%A6%E6%A1%88%E4%BE%8B.png)

#### 圖5：測試案例圖

#### 5.結論與未來展望 (Conclusion and Future Prospects)

5.1 結論 

本專題成功開發了一套整合物聯網 (IoT) 與電腦視覺技術的「智慧停車場管理系統」。透過 ESP32 微控制器作為感知末端，結合 Python Flask 後端伺服器與 MySQL 資料庫，我們實現了從車輛進出、車牌辨識、計費試算到違規偵測的完整自動化流程。



(1) 解決無障礙車位管理痛點： 傳統停車場對於身障車位的管理多依賴人工巡視，效率低且無法即時制止。本系統提出的「即時影像違規偵測機制」，能透過車牌白名單比對，在非授權車輛停入的當下立即觸發聲光警示，有效達到嚇阻作用，保障身障人士的合法權益。


(2) 高可靠度的軟硬體整合： 在實作過程中，我們克服了超音波感測器訊號抖動的問題，設計了「時間視窗濾波演算法」，大幅降低了誤判率。經系統測試驗證，車位狀態偵測的準確度達 95% 以上，且車牌辨識與開閘反應時間控制在 2 秒內，符合即時系統之需求。


(3) 完整的系統架構實踐： 本專題涵蓋了客戶端 App 開發、伺服器 API 設計、資料庫正規化以及地端硬體電路控制。這不僅驗證了我們在軟體工程課程中所學的理論，更展現了將不同技術領域（嵌入式系統、網頁後端、行動應用）進行系統化整合的能力。


5.2 未來展望 


(1) 導入深度學習技術： 目前採用的 OpenCV 傳統影像處理技術（如 Tesseract OCR），在光線不足或車牌髒污時辨識率會下降。未來可導入 YOLO (You Only Look Once) 或 CRNN 等深度學習模型，提升在複雜環境下的車牌定位與辨識能力，甚至能辨識車輛顏色與型號。


(2) 多元支付串接： 目前的繳費功能僅為模擬試算。未來可嘗試串接第三方支付 API（如 Line Pay、ECPay 綠界科技），實作真實的金流交易，讓使用者能透過 App 直接完成付款，實現真正的「無現金、無感出場」。


(3) 即時通訊推播： 目前的違規警示僅限於現場聲光。未來可結合 Line Notify 或 Telegram Bot API，當偵測到無障礙車位違規或設備異常（如斷線）時，主動推播訊息至管理員的手機，讓管理人員能第一時間遠端掌握狀況。


(4) 大數據分析與預測： 隨著系統運作時間增加，累積的進出紀錄將成為珍貴數據。未來可開發資料分析模組，分析尖峰時段與週轉率，甚至結合機器學習預測剩餘車位數，為使用者提供更精準的車位導引服務。
