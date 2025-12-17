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

#### 2. 系統模組劃分 (Module Decomposition)
為了方便分工開發，將系統劃分為以下四大核心模組：

##### 2.1 硬體控制模組 (Hardware Control Module)
功能： 直接控制電子元件。

子模組：

感測子系統： 讀取超音波距離數據，判斷車位是否有車。

執行子系統： 控制伺服馬達 (閘門開關)、控制 LED/蜂鳴器 (違規警示)。

通訊子系統： 透過 Wi-Fi (HTTP/MQTT) 將狀態傳送至後端。

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

    erDiagram
         Users ||--o{ Vehicles : "擁有多輛車"
         Vehicles ||--o{ Records : "產生進出紀錄"
         Vehicles ||--o{ Violations : "造成違規"
         ParkingSpots ||--o{ Violations : "發生於"

    Users {
        int user_id PK
        string name "姓名"
        string phone "電話"
        boolean is_disabled "是否身障人士(白名單)"
    }

    Vehicles {
        string plate_number PK "車牌號碼"
        int user_id FK "車主ID"
    }

    ParkingSpots {
        string spot_id PK "車位編號(A01)"
        string type "類型(一般/無障礙)"
        string status "狀態(空/滿/維修)"
    }

    Records {
        int record_id PK
        string plate_number FK
        datetime entry_time "進場時間"
        datetime exit_time "出場時間"
        int fee "費用"
        boolean is_paid "是否繳費"
    }

    Violations {
        int violation_id PK
        string plate_number FK
        string spot_id FK
        datetime detected_time "偵測時間"
        string image_path "存證照片路徑"
    }
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

流程圖 (Flowchart): 

    flowchart TD
    Start((開始)) --> SensorTrigger[硬體回報: 車位被佔用]
    SensorTrigger --> Delay[延遲 3秒: 等待車輛停妥]
    Delay --> Capture[觸發相機拍照]
    Capture --> LPR[執行 OpenCV 車牌辨識]
    
    LPR --> CheckPlate{辨識是否成功?}
    CheckPlate -- 否/模糊 --> NotifyAdmin[通知管理員人工確認]
    NotifyAdmin --> KeepMonitor[維持監控]
    
    CheckPlate -- 是 --> QueryDB[查詢 Users 白名單資料表]
    QueryDB --> IsWhiteList{是否為身障車牌?}
    
    IsWhiteList -- 是 (合法) --> LogNormal[記錄正常停車]
    LogNormal --> GreenLight[指令: 亮綠燈/無動作]
    
    IsWhiteList -- 否 (違規) --> LogViolation[寫入 Violations 資料表]
    LogViolation --> SaveImg[儲存違規照片佐證]
    SaveImg --> Alert[指令: 觸發蜂鳴器 & 紅燈]
    Alert --> End((結束))
    GreenLight --> End
    
詳細步驟說明：

事件觸發： 接收到硬體確認有車停入 (is_occupied=true)。

影像擷取與前處理：

使用 Python cv2.VideoCapture 抓取影像。

影像處理演算法： 高斯模糊 (降噪) -> 轉灰階 -> Canny 邊緣偵測 -> 尋找輪廓 (Contours) -> 切割出車牌區域。

OCR 識別： 將切割圖片丟入 Tesseract 或專用模型，轉出字串 (例如 "ABC-1234")。

邏輯比對：

SELECT is_disabled FROM Users WHERE plate_number = 'ABC-1234'

若回傳 True：合法。

若回傳 False 或 NULL (查無此人)：視為違規。

動作執行： 根據結果呼叫對應的 API 更新現場燈號。

#### 4. 模組封裝建議 (Implementation Suggestions)
為了讓程式碼整潔，建議在詳細設計階段就定義好 Python 的 Class 結構：

class HardwareController: 負責處理 MQTT/HTTP 訊號，不含業務邏輯。

class LPRService: 封裝 OpenCV 功能，輸入圖片，輸出字串。

class ParkingManager: 核心邏輯層，負責呼叫資料庫並決定「是否違規」。

### 測試計畫書
#### 1. 測試目標與範圍 (Objectives & Scope)
##### 1.1 測試目標
驗證軟硬體整合的連線穩定性（ESP32 與 Python Server 溝通）。

確認核心功能（車牌辨識、計費、違規偵測）邏輯正確無誤。

評估系統反應速度是否符合需求規格書中的性能要求（如：辨識 < 2秒）。

##### 1.2 測試範圍
模組測試： 硬體感測器讀數、影像辨識演算法、資料庫存取函式。

整合測試： API 資料傳輸、前後端資料同步。

系統測試： 完整的使用者情境（從進場到出場）。

例外測試： 網路斷線、車牌髒污無法辨識、非白名單佔用車位。

#### 2. 測試策略與方法 (Test Strategy)
我們採用 由下而上 (Bottom-Up) 的測試策略，先確保零件沒問題，再測整體流程。

##### 2.1 單元測試 (Unit Testing)
針對獨立的程式碼或硬體元件進行測試。

硬體端：

使用 Arduino Serial Monitor 檢查超音波感測器是否能準確回傳距離（誤差 < 5cm）。

測試伺服馬達 (Servo) 是否能準確轉動 90 度（開閘）與 0 度（關閘）。

軟體端 (Python)：

測試 calculate_fee(entry_time, exit_time) 函式，輸入不同時間，檢查金額是否正確。

測試資料庫連線函式，確保能正確 INSERT 和 SELECT 資料。

##### 2.2 整合測試 (Integration Testing)
重點在於「介面 (Interface)」的溝通。

硬體與後端： 觸發 ESP32，檢查 Server 是否收到 JSON 請求 (POST /api/hardware/update)。

後端與影像處理： 傳送一張測試照片給辨識模組，檢查是否能回傳正確車牌字串。

後端與資料庫： 模擬一筆違規資料，檢查資料庫 Violations 表格是否新增該筆紀錄。

##### 2.3 系統測試 (System Testing) - 驗收測試
模擬真實場景的端對端測試 (End-to-End)。

情境 A (正常停車)： 車輛進場 -> 閘門開啟 -> 停入一般車位 -> App 顯示車位減少。

情境 B (無障礙違規)： 非白名單車輛停入無障礙車位 -> 系統偵測 -> 現場發出警報 -> 後台記錄違規。

#### 3. 測試環境配置 (Test Environment)
![03](https://github.com/11224204lbt/parking-system/blob/main/2.jpg)
#### 4. 測試案例設計 (Test Cases)
以下列出最具代表性的三個測試案例 (Test Cases)。你在報告中可以列出這些表格，證明你有嚴謹的測試。

##### 4.1 案例 ID: TC-01 - 車輛進場與辨識
前置條件： 入口無車，閘門關閉，資料庫正常運作。

測試步驟：

將玩具車（貼有車牌 ABC-1234）推至入口感測區。

觀察序列埠 (Serial Monitor) 是否觸發拍照。

觀察閘門是否開啟。

預期結果：

伺服器 Log 顯示「辨識成功：ABC-1234」。

資料庫 Records 表新增一筆入場紀錄。

閘門伺服馬達轉動至開啟狀態。

##### 4.2 案例 ID: TC-02 - 無障礙車位違規偵測 (核心功能)
前置條件： 車牌 XYZ-999 不在身障白名單 (Users) 中。

測試步驟：

將車輛 XYZ-999 停入無障礙車位感測範圍。

持續停留超過 5 秒（模擬停妥）。

預期結果：

後端判斷該車牌非白名單。

現場蜂鳴器響起，紅燈閃爍。

資料庫 Violations 表新增一筆違規紀錄，並附帶照片路徑。

##### 4.3 案例 ID: TC-03 - 網路異常處理 (壓力/例外測試)
前置條件： 系統運作中。

測試步驟：

將伺服器網路斷開 (或關閉 Wi-Fi 分享器)。

觸發 ESP32 感測器。

預期結果：

ESP32 不應當機 (Crash)。

ESP32 應嘗試重新連線 (Reconnecting...) 或亮起故障燈號。

#### 5. 缺陷管理與通過標準 (Defect Management & Criteria)
##### 5.1 嚴重性分級 (Severity)
如果在測試中發現 Bug，依據以下等級分類：

Critical (嚴重)： 系統崩潰、無法進場、資料庫資料遺失。 -> 必須修復才能展示。

Major (主要)： 辨識率低於 70%、警示燈號延遲超過 5 秒。 -> 建議修復。

Minor (次要)： App 介面跑版、文字拼寫錯誤。 -> 行有餘力再修。

##### 5.2 通過標準 (Pass Criteria)
專題要能夠上台演示 (Demo)，需滿足：

所有 Critical 等級 Bug 已修復。

車牌辨識在展示環境下（燈光充足）成功率達 90%。

無障礙違規警示功能 100% 觸發成功。

#### 6. 資源配置與時程 (Resources & Schedule)
測試人員： 組員 A (硬體測試), 組員 B (軟體/API 測試)。

時程規劃：

第 1 週： 單元測試 (確保每個感測器和 API 都是活的)。

第 2 週： 整合測試 (把 ESP32 接上 Server，跑通流程)。

第 3 週： 系統測試 (實際拿車子跑，調整辨識參數)。

Demo 前 3 天： 凍結程式碼 (Code Freeze)，不再加新功能，只修 Bug。
