# 使用 Docker 建置 Apache NiFi 服務，並實作透過 NiFi 建立 Data pipeline
* 主要參考 **hueiyuan** 的 "IT 30 天鐵人賽" 文章
* iThome目錄：https://ithelp.ithome.com.tw/articles/10265057
* GitHub: https://github.com/hueiyuan/nifi-it30/tree/main
* 我的筆記：https://hackmd.io/@wilson330/NiFi_implementation



# Docker
Docker 是一種軟體平台，可讓您快速地建立、測試和部署應用程式。Docker 將軟體封裝到名為容器的標準化單位，其中包含程式庫、系統工具、程式碼和執行時間等執行軟體所需的所有項目。  

## Step1: 安裝Docker & Docker Compose
進入官網，選擇對應的OS版本，下載**Docker** (windows為Docker Desktop)，以下是Windows的下載連結：
https://docs.docker.com/desktop/install/windows-install/

下載後註冊帳號、重啟後，Docker Desktop即可正常運行。  

* Docker Compose已經整合在Docker Desktop中，使用Windows則無需另外安裝，Linux則要安裝插件。


## Step2: 使用Docker建置Apache nifi服務

### 至欲執行的目錄位置建立所需檔案 
* `.env` : 放  NiFi 啟動之後需要登入的帳號密碼以及使用AWS服務所需的key
* `requirements.txt` : 所需安裝的套件 (注意版本衝突問題)
* `Dockerfile` : 建置新的image所需的script
* `docker-compose.yaml` : 建立 NiFi 和 Nifi-registry 服務 (建完image後使用)

要透過Docker運行NiFi，需要先安裝 **Apache NiFi Docker Image**  
於 Command prompt 輸入指令 `docker pull apache/nifi` 來建立 Image

所需的檔案建立好後，執行指令來建置新的image (**注意Docker Desktop需開啟**)  
```
docker build -t nifi-sample .
```

* 可以透過指令`docker images`來檢查已建立之image  

建置完成image後，執行指令建立NiFi服務並啟動container  
```
docker-compose up -d
``` 

* 啟動container `docker compose start`  
* 關閉container `docker compose stop`  

### ERROR處理
* 安裝時遇到 "Unexpected WSL error"：沒特別設定的話，windows的預設就是不自動更新WSL(Windows Subsystem for Linux)的服務，可以使用 Command prompt，輸入指令 `wsl --status` 查詢，並輸入 `wsl --update` 來進行安裝。重啟Docker Desktop後應該就沒問題了！
* `requirements.txt`中的`PyGObject`套件安裝有問題 -> 直接拿掉
* Docker 執行NiFi失敗 -> 在`docker-compose.yaml`中加入環境變數`NIFI_SENSITIVE_PROPS_KEY`，確保有12字元長以上
(參考這篇 https://stackoverflow.com/questions/69217062/docker-nifi-1-14-0-startup-failure-caused-by-org-apache-nifi-properties)
* localhost不是私人連線(非HTTPS)，參考這篇的方法2 https://forum.gamer.com.tw/C.php?bsn=60030&snA=599497
* 使用者密碼應該也要**符合12字元長以上**的規則，可以是亂碼，否則會無法登入

### 檔案處理
複製 Container 內的檔案到 Host  
`docker cp <containerID>:/file/path/within/container /host/path/target`

進入 Container terminal  
`docker exec -it <containerID> bash`

# Apache NiFi
Apache NiFi是一個用來建立 Data Pipeline 的 opensource，由於目前大環境的現況，在實際應用中我們會很常面臨到需要做大量資料轉換的場景，可能要整合多個 Datasource 然後做整併處理，來給商業端或是ML相關的下游做應用，所以 Data Pipeline 在真實場景上其實是扮演著一個非常重要的角色。  

## NiFi 與設定有關的component
#### 1. Controller Service
只要與第三方的平台、cloud 或 DB 等都需要透過Controller Service 來做一個對接，才可以將我們要的資料來做一個輸入或是輸出。  
如果同時有多個 Data Pipieline 都需要對同一個 DB 或是一個 Datasource 建立多個連線的話，則對於另一端可能會造成不可預期的影響。所以若能透過同一一個 Object 來建立其中的連線的話，對於 Pipeline 的處理一方面更加單純，另一方面對於提供資料的那一端也不會有太多不必要且重複的網路連線。

* 對AWS Cloud建立連線的設定
* 讀取或寫入特定 format 的設定

## (實作) 建置data pipeline
Ref:  
https://ithelp.ithome.com.tw/articles/10280344

### 進入NiFi Web UI
確認Docker container成功運行後，可以透過網址進入NiFi服務  
* **NiFi** : https://localhost:8443/nifi
* **NiFi-Registry** : http://localhost:18080/nifi-registry

### 建立 data pipeline
* 讀取儲存在 AWS S3 中的 csv檔
* 將 "stage" 欄位的字串改為數值，並保留 "Number", "Digimon", "Stage", "Type" 等四個欄位
* 更新 flowfile 的 attribute，並將經過處理的 flowfile 儲存成新的檔案
* (Exceed) 將處理後的資料檔案由container傳至本機

![pipeline](/pipeline.PNG)


# Amazon Web Services (AWS)
提供多項服的雲端平台，如資料倉儲、資料湖等。

## Simple Storage Service (S3)
* 一種物件儲存服務，提供領先業界的可擴展性、資料可用性、安全性及效能。
* 建置資料湖，可存放各式型態資料。

### 使用說明
1. 辦理AWS帳戶 (需信用卡資訊，新會員有一年免費方案)
2. 進入主控台，選擇S3服務
3. 點選 "**建立儲存貯體**"
4. 參考 https://ithelp.ithome.com.tw/articles/10253077
5. 進入 "**安全憑證**" 找到 "**存取金鑰**" 並建立

### 使用注意
* 非GET的API每月僅限2000次，記得不要過度使用 (如LIST)
