# 第七天：想想我們為何而來：一個完整DevOps開發流程的十二個要素

Author: Nick Zhuang
Type: DevOps

# 前言

前六天我們簡單介紹了Docker和k8s的架構，今天我們來稍微聊一下整個DevOps的開發流程，透過全貌從另外一個新的面向去了解k8s，這篇主要偏開發上的流程概念，中間的部分流程會用到k8s，我想就某種程度而言，這就已經代表了完整的DevOps的流程了吧，不過因為我們受限於時空地理的限制，不見得每一段都會仔細接觸過，但個人看完這相關的資料，覺得蠻重要的，所以來分享下。

# 十二要素

- [CodeBase](#CodeBase)
- [Dependencies](#Dependencies)
- [Config](#Config)
- [Backing services](#Backing-services)
- [Build, release, run](#Build-release-run)
- [Processes](#Processes)
- [Port binding](#Port-binding)
- [Concurrency](#Concurrency)
- [Disposability](#Disposability)
- [Dev/prod parity](#Devprod-parity)
- [Logs](#Logs)
- [Admin processes](#Admin-processes)

# CodeBase

### 一份基準代碼（Codebase），多份部署（deploy）

這個部分是指基準代碼，一般我們在做開發的時候，當有代碼要做版本管控的時候，我們可能會用到像是[Git](https://git-scm.com/)、[Mercurial](https://www.mercurial-scm.org/)、[Subversion](http://subversion.apache.org/)等等的工具，我們也會在本地端架設repository（簡稱repo），也就是代碼數據庫的副本。

我們來看張圖：

![https://ithelp.ithome.com.tw/upload/images/20190922/20120468Sa07rHBlZl.png](https://ithelp.ithome.com.tw/upload/images/20190922/20120468Sa07rHBlZl.png)

基準代碼和應用之間必須保持一對一的關係：

- 如果對應多個基準代碼（Codebase），那他就不能算是一個應用（app），而是一個分布式系統
- 多個應用（app）不應共享一份基準代碼，必須將共享代碼拆分並以[依賴](#Dependencies)去管理

# Dependencies

### 明確的宣告並隔離依賴（dependency）

這個部分是指在開發環境的設置中，定義一個相依性的套件集合，這個套件集合是用來幫助開發用的，套件集合裡面定義了彼此之間的關係。

主要的特點是說，在應用程序運行的途中，只要是用到相依性的套件，那必然是有定義過的，系統工具也是同樣的邏輯，也就是明確地宣告並隔離依賴。

這樣的好處是說，當我們要Debug的時候，當本身應用程序發生問題的時候，我們可以透過這個明確宣告的依賴關係清單去查詢用到了哪些套件，舉個例子：

Debug流程

1. 設定中斷點
2. 再次執行，尋找問題點
3. 問題可能是語法錯誤、執行階段錯誤、邏輯錯誤
4. 若程式碼無誤，有可能相依性套件有問題
5. 查詢依賴清單，排查可能有問題的套件

所以可以看到，如果沒有依賴清單或是依賴性是隱性的話排查問題會非常麻煩。

# Config

### 在環境中存儲配置

這個部分是說關於環境中設置的問題，不是指一般程序使用的設定檔，這種設置是環境變量上的，像是我們在linux下env指令得到的類似結果，他會推薦使用這種方式是因為這樣可以針對不同的使用情境切換不同的變數值，而不會受到設定檔的限制，設定檔的限制會有：

- 有可能將不同使用情境的設定檔混在版本控管環境中的風險
- 不同的程式語言設置方式不同，彈性不足
- 當設定檔的配置改動的時候，有可能需要改代碼

使用環境變數的設置就不會有這樣的風險，另外如果變數彼此間獨立性夠強，當應用程序不斷擴展，需要更多種類的部署時，這種配置管理方式就能保有各種部署上的特性。

# Backing services

### 把後端服務( backing services )當作附加資源

後端服務是指程序運行所需要的通過網絡調用的各種服務，如數據庫（MySQL），消息/隊列系統（RabbitMQ），SMTP郵件發送服務（Postfix），以及緩存系統（Memcached）。

在這樣的原則下，應用（app）不會對不同地方提供的服務而有所區別，一般來說，任意的[部署](#CodeBase)，都應該可以在不進行任何代碼改動的情況下，將本地MySQL數據庫換成第三方服務。

我們來看張圖：

![https://ithelp.ithome.com.tw/upload/images/20190922/20120468HbVXjsYgTp.png](https://ithelp.ithome.com.tw/upload/images/20190922/20120468HbVXjsYgTp.png)

部署可以按需加載或卸載資源。

在這樣的設定下，當數據庫有問題的時候，只要切換做好就可以了，不需要去改代碼。

# Build, release, run

### 嚴格分離建置和運行步驟

[基準代碼](#CodeBase)轉化為一份部署(非開發環境)需要經過以下三個階段：

- 建置：與[依賴](#Dependencies)相關，是將代碼轉成執行檔的過程
  - 開發人員可以增加些建置流程，方便Debug，節省開發成本
- 發布：與[配置](#Config)相關，結合當前建置與部署的結構，可在實際環境運行
  - 建立release以及rollback的機制，並控制版本號
- 運行：與[進程](#Processes)相關，針對發佈的版本，在實際環境啟動運行一系列的應用（app）
  - 盡量不要在運行階段改code，這樣會混淆到建置的步驟
  - 這個階段不一定需要人為觸發，可以自動進行，也因為如此，會需要保持較少的模塊

# Processes

### 以一個或多個無狀態進程運行應用

在運行環境中，應用程序通常是以一個或多個進程運行的。

這些進程必須是沒有狀態的且彼此之間並沒有共享資源。任何需要持久化的數據都要存儲在[後端服務](#Backing-services)內，比如數據庫。

# Port binding

### 通過端口綁定( Port binding )來提供服務

互聯網應用有時會運行於服務器的容器之中。

**這些應用必須要能夠自我加載**而不依賴於任何網絡服務器就可以創建一個網絡服務。互聯網的應用會**通過端口綁定來提供服務**，並監聽發送至該端口的請求。

# Concurrency

### 通過進程模型進行擴展

任何計算機程序，一旦啟動，就會生成一個或多個進程。

**在應用中，這些進程的運行**主要藉鑑於[unix守護進程模型](https://adam.herokuapp.com/past/2011/5/9/applying_the_unix_process_model_to_web_apps/)，可以運用這個模型去設計應用架構。

有兩個面向：

- 進程類型：依照不同的工作種類做區分
- 進程構成：各種不同數量的進程類型所組成的進程集合

參考下圖：

![https://ithelp.ithome.com.tw/upload/images/20190922/20120468s0F6QJjfO7.png](https://ithelp.ithome.com.tw/upload/images/20190922/20120468s0F6QJjfO7.png)

因為進程的無共享及水平分區的特性，上述進程模型會在系統擴展時有顯著的效用。

# Disposability

### 藉由快速啟動和終止來最大化應用的健壯性

**應用的進程應該要是*易處理（disposable）*的，意思是說它們可以瞬間開啟或停止，並追求最小的啟動時間。**

理想狀態下，進程從敲下命令到真正啟動並等待請求的時間應該只需很短的時間。更少的啟動時間提供了更敏捷的[發布](#Build-release-run)以及擴展過程，此外還增加了健壯性，因為進程管理器可以在授權情形下容易的將進程搬到新的物理機器上。

# Dev/prod parity

### 盡可能的保持開發，預發布，線上環境相同

**應用若要做到[持續部署](http://avc.com/2011/02/continuous-deployment/)就必須縮小本地與線上差異。**
如下所述：

- 縮小時間差異：開發人員可以快速部署代碼
- 縮小人員差異：開發人員應該密切參與部署過程以及代碼在線上的表現
- 縮小工具差異：盡量保證開發環境以及線上環境的一致性

將上述總結變為一個表格如下：

|                    |  傳統應用  | 12-factor應用 |
|--------------------|---------- |--------------|
|     每次部署間隔     | 數週       |  幾小時       |
| 開發人員 vs 運維人員  | 不同的人   |  相同的人      |
| 開發環境 vs 線上環境  | 不同       |  盡量接近     |

應用的開發人員應該反對在不同環境間使用不同的後端服務，即使適應器已經可以幾乎消除使用上的差異。這是因為，不同的後端服務意味著會突然出現的不兼容，從而導致測試、預發布都正常的代碼在線上出現問題。這些錯誤會給持續部署帶來阻力。從應用程序的生命週期來看，消除這種阻力需要花費很大的代價。

不同後端服務的適應器仍然是有用的，因為它們可以使移植後端服務變得簡單。但應用的所有部署，這其中包括開發、預發布以及線上環境，都應該使用同一個後端服務的相同版本。

# Logs

### 把日誌當作事件串流

*日誌*使得應用程序運行的動作變得透明。在基於服務器的環境中，日誌通常被寫在硬盤的一個文件裡，但這只是一種輸出格式。

應用本身不需考慮存儲自己的輸出流，且不該試圖去寫或者管理日誌文件。相反，每一個運行的進程都會直接的標準輸出（stdout）事件流。開發環境中，開發人員可以通過這些數據流，實時在終端看到應用的活動。

這些事件流可以輸出至文件，或者在終端實時觀察。最重要的，輸出流可以發送到[Splunk](http://www.splunk.com/)這樣的日誌索引及分析系統，或[Hadoop/Hive](http://hive.apache.org/)這樣的通用數據存儲系統。這些系統為查看應用的歷史活動提供了強大而靈活的功能，包括：

- 找出過去一段時間特殊的事件。
- 圖形化一個大規模的趨勢，比如每分鐘的請求量。
- 根據用戶定義的條件實時觸發警報，比如每分鐘的報錯超過某個警戒線。

# Admin processes

### 後台管理任務當作一次性進程運行

[進程構成](#Processes)（process formation）是指用來處理應用的常規業務（比如處理web請求）的一組進程。與此不同，開發人員經常希望執行一些管理或維護應用的一次性任務，例如：

- 運行數據移植。
- 運行一個控制台，來執行一些代碼或是針對線上數據庫做一些檢查。
- 運行一些提交到代碼倉庫的一次性腳本。

# 小結

我們看完了以下12個重要的應用組成要素，發現到可以分為幾個面向：

![https://ithelp.ithome.com.tw/upload/images/20190922/20120468f1d7FDwNnW.png](https://ithelp.ithome.com.tw/upload/images/20190922/20120468f1d7FDwNnW.png)

- Unify（單位化）：CodeBase、Processes
- Specify（區隔化）：Dependencies、Config、Backing services、Build, release, run
- Conform（協調化）：Port binding、Concurrency、Admin processes
- Auto（敏捷化）：Disposability、Logs
- Sync（同步化）：Dev/prod parity

## Unify（單位化）

這個部分是指定義所謂應用中的最小單位，定義了最小的單位以後就能夠方面我們去拆分其中的結構

所以可以看到，這邊我就把CodeBase（基準代碼）、Processes（進程）視為一類，因為就某些方面來說，他們確實構成了整個應用（app），也影響了建置、部署等等階段的結構。

## Specify（區隔化）

這個部分是指盡可能地將結構做拆分，能夠隔離的、切開來看的就盡量區分，盡量不要有重疊的部分，這主要是強化元件中的獨立性，這樣在分開開發或是除錯上都會有幫助，有幾個元件：Dependencies（依賴）、Config（配置）、Backing services（後端服務）、Build, release, run（建置, 發布, 運行）

## Conform（協調化）

這個部分是指元件組合上的協調度，如何有效的組合元件又不會出問題，而且是當這樣的組合是有必要性的時候，我們可以看到這些組合是為了增加應用上的拓展性，如：Port binding（端口綁定）、Concurrency（並發）、Admin processes（管理進程）

## Auto（敏捷化）

這個部分是指在單個元件上，它是否能做到自動啟動一些機制，以達到幫助維運的需求，像是自動紀錄、自動快速開啟與結束，如：Disposability（易處理）、Logs（日誌）

## Sync（同步化）

這個部分是指不同環境上的轉移、等價等等，這裡的Dev/prod parity（開發環境與線上環境等價）就是滿足這樣的需求設計的，或許將來會有新的性質會新增到這個項目當中

## 總結

看完了這5個面向，讀者是不是對於這個12要素更了解些呢？其實一言以貫之，我們可以看到，這些要素的相關性及連結，是利用以下邏輯去思考的：

- 定義出最小單位
- 依照最小單位盡可能的拆分、拆解，直到變成不能再拆解的狀態為止
- 拆解好的元件，透過一些重新組合，定義了新的元件
- 接著，針對不同元件的特性作解析
- 最後，比較不同元件的相似度

所謂的12要素（12-factor）大致上就是在描述這些東西，希望針對這一系列的介紹能優化大家對於在開發流程的觀念，就好比是有好的coding-style一樣，相信針對這部份的介紹也能讓大家理解到k8s的重要性，我們明天見！

# 參考資料

- [構成應用（app）的12個要素：12-factors](https://12factor.net)