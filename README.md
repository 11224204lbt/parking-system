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
1. 系統架構設計 (System Architecture Design)
本系統採用 B/S (Browser/Server) 與 Client/Server 混合架構，整體邏輯分層如下：

1.1 邏輯架構圖 (Logical Architecture) - 三層式架構
系統分為 感知層 (Perception/Client)、業務邏輯層 (Business Logic) 與 資料層 (Data)。

感知層 (Client Tier & IoT Edge)：

硬體端： ESP32/Arduino + 超音波感測器 + 攝影機 + 閘門伺服馬達 + 警示燈/蜂鳴器。負責現場資料蒐集與物理控制。

軟體端： App Inventor (使用者端 APP)、Web 瀏覽器 (管理員後台)。負責人機互動。

業務邏輯層 (Application Tier)：

後端伺服器 (Backend Server)： 執行 Python (Flask/Django) 或 Node.js。負責接收硬體訊號、執行 OpenCV 影像辨識、邏輯判斷 (如：是否違規佔用)、API 回應。

資料層 (Data Tier)：

資料庫： MySQL。負責儲存使用者資料、停車紀錄、違規影像路徑與費率設定。

1.2 實體部署架構 (Physical Architecture)
地端 (Local): 停車場入口閘門控制器、無障礙車位監測模組 (ESP32)。

雲端/伺服器端 (Server): 部署於實驗室伺服器或雲端 (如 AWS/GCP)，運行 Web Server 與 Database。

2. 系統模組劃分 (Module Decomposition)
為了方便分工開發，將系統劃分為以下四大核心模組：

2.1 硬體控制模組 (Hardware Control Module)
功能： 直接控制電子元件。

子模組：

感測子系統： 讀取超音波距離數據，判斷車位是否有車。

執行子系統： 控制伺服馬達 (閘門開關)、控制 LED/蜂鳴器 (違規警示)。

通訊子系統： 透過 Wi-Fi (HTTP/MQTT) 將狀態傳送至後端。

2.2 影像處理模組 (Image Processing Module)
功能： 處理進出場與車位監控的影像分析。

子模組：

車牌辨識 (LPR Core)： 接收影像 -> 灰階化/二值化 -> 車牌定位 -> 字元分割 -> OCR 辨識 -> 輸出車牌字串。

違規偵測 (Violation logic)： (針對無障礙車位) 當硬體感測到有車，但車牌辨識結果不在「身障白名單」內，標記為違規。

2.3 後端管理模組 (Backend Management Module)
功能： 系統的大腦，處理所有邏輯運算。

子模組：

計費引擎： 計算 (出場時間 - 入場時間) * 費率。

車位狀態管理： 接收硬體訊號，更新資料庫中的 status (空/滿/違規)。

API 介面： 提供 RESTful API 供 App 和硬體呼叫。

2.4 前端互動模組 (Frontend Interaction Module)
功能： 提供視覺化介面。

子模組：

管理員儀表板 (Web)： 顯示即時車位圖、營收報表、強制開閘按鈕。

使用者 APP (App Inventor)： 查詢剩餘車位、試算停車費。

3. 介面設計 (Interface Design)
3.1 外部介面 (External Interface) - API 設計
定義硬體與軟體、前端與後端溝通的標準。採用 HTTP RESTful 風格，回傳格式為 JSON。
![03](https://github.com/11224204lbt/Chatgpt/blob/main/1.png)
