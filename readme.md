---
layout: post
title: "使用 jamf Pro 自動佈署 802.1x "
date: 2019-11-28 11:00:00
image: '![cover_photo](https://raw.githubusercontent.com/henry510859/TWDC_blog_photo/master/802.1x%20%E8%87%AA%E5%8B%95%E4%BD%88%E7%BD%B2/cover%20photo.jpg)'
description:
category: 'network'
tags:
- jamf Pro
- network
introduction:
---

# IEEE 802.1X 是什麼？
---
802.1X是IEEE制定的一項身分驗證標準，而非單一的網路通訊協定，在802.1X的架構下，企業可以透過前端的網路設備，像是交換器、無線AP等，要求使用者輸入連線所需要的一組帳號、密碼，向後端的帳號伺服器發出驗證的請求，確認使用者具備存取網路資源的權限之後，前端的設備就會開啟網路埠（無線網路的網路埠是虛擬的），允許連線通過。

802.1X 驗證分為三個基本部分  
  1. **客戶端** 在 Wi-Fi 工作站上所執行的軟體用戶端。
  2. **設備端** Wi-Fi 存取點。
  3. **驗證伺服器** 驗證資料庫，通常會是 Cisco Secure Access 或 freeradius 等 radius 伺服器。

![802.1x 示意圖](https://raw.githubusercontent.com/henry510859/TWDC_blog_photo/master/802.1x%20%E8%87%AA%E5%8B%95%E4%BD%88%E7%BD%B2/802.1x.png)

當客戶端要連線到網際網路的時候必須輸入正確的帳號密碼，設備端會向 RADIUS server 詢問帳號密碼是否正確以及該帳號在內網中的權限為何，當認證通過後客戶端就可以依據當時所設定的權限來進行連網的動作，RADIUS server 也可以結合 Active Directory 或是 LDAP server 來進行更進一步的控管。

## 802.1x 網路建置
想知道要如何快速的建置 802.1x 企業網路嗎？
只要照著下列步驟就可以完成囉

### RADIUS server 設定
我們使用 RT2600ac 來做示範，為什麼選擇 RT2600ac?
因為在 RT2600ac 裡的作業系統 SRM 讓管理者可以非常簡單快速的建置 802.1x 企業網路。

 ▼ 打開 RT2600ac 管理頁面
![ASM](https://raw.githubusercontent.com/henry510859/TWDC_blog_photo/master/802.1x%20自動佈署/Synology%20router%20管理介面.png)
 ▼ 點選套件中心
 ![ASM](https://raw.githubusercontent.com/henry510859/TWDC_blog_photo/master/802.1x%20自動佈署/套件中心.png)
▼ 選擇 RADIUS server 下載  
▼ 下載完成後點開打開 RADIUS server 應用程式  
▼ 這邊可以選擇驗證的方式，如果選擇 **LDAP 使用者** 或 **網域使用者** 務必先讓 RT2600ac **加入LDAP 伺服器或是網域中**  
▼ 點選用戶端，這邊用戶端設定並不是建立使用者帳號密碼而是給 **Wi-Fi 存取點** 使用  
▼ 點選新增，設定名稱，這邊建議設定成好辨識的名稱  
▼ 共用金鑰為等等設定 **WPA2 enterprise** 所使用  
▼ IP 位址設定成 **Wi-Fi 存取點** 的 IP 位址  
這樣 RADIUS server 的設定就完成囉  

### SSL 憑證

SSL 憑證設計一組編碼連結於客戶端與設備端之間，在解密以前，資料在網路之間傳送時是無法被讀取的，SSL憑證保護這些資料，以避免被其它網路的某些人監控，因為他們無法知道已經加密的資料。

在 RADIUS server 上設定好憑證是非常重要的，因為安全的憑證是要付費的，建議 **不要自己簽憑證** ，一定要去購買才能確保網路安全。
#### 建立私鑰以及 CSR  
我們使用OpenSSL來建立所需要的憑證
1. 建立一組私鑰給主機

  `` openssl genrsa -out server.key 4096``

2. 建立 CSR (Certificate Signing Request)
CSR 又叫作憑證簽名申請，在跟 SSL 廠商申請憑證會需要。

``openssl req -new -key server.key -sha256 -out server.csr``

在產生 CSR 的時候 OpenSSL 會要求輸入一些資訊

```Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, YOUR name) []:
Email Address []:
A challenge password []:
An optional company name []:
```
這些資訊非常重要，請謹慎填寫。

#### 建立 CA  
因為教學方便，我們自行建立 CA 來簽署憑證
1. 建立 CA 的私鑰

``openssl genrsa -des3 -out ca.key 4096``

2. 建立 CA 所需要的憑證，這邊要特別注意 CA 的日期，如果過期了過去所簽署過的所有憑證都要重來。

``openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.cer``

#### CA 簽署 CSR 來產生憑證  
CA 建立好之後就可以用來簽署上面建立的 csr

  ``openssl x509 -req -in server.csr -CA ca.cer -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256``

### 憑證匯入  
當憑證建立完成後要匯入到 RT2600ac 裡面  
▼ 打開 RT2600ac 管理頁面  
▼ 點選控制台  
▼ 點選服務  
▼ 點選憑證  
▼ 點選匯入憑證  
▼ 在私鑰上傳製作的主機私鑰  
▼ 在憑證上傳簽署的憑證  
▼ 在中繼憑證上傳 CA 的憑證  

### 設定 WPA2 enterprise  
要設定WPA2 enterprise之前有兩個條件，缺一不可
1. RADIUS server
2. 憑證

當以上都設定完成，就可以開始設定 WPA2 enterprise

▼ 打開 RT2600ac 管理頁面  
▼ 點選Wi-Fi Connect  
▼ 點選無線網路  
▼ 安全層級選擇 WPA2-Enterprise  
▼ 輸入 RADIUS server IP位址  
▼ 連接阜編號填寫1812  
▼ 共用金鑰填寫在 RADIUS server 用戶端的共用金鑰  
▼ 完成後點選套用，套用後務必把 RT2600ac **重新開機**  

這樣就可以順利的使用802.1x 網路囉  
想知道更進階的自動佈署就繼續看下去吧  

## jamf Pro 自動佈署

在 jamf Pro 裡面我們可以設定讓裝置自動的連上我們剛才所建立的802.1x網路  
首先要先把憑證匯入到 jamf Pro 裡面  
但是 jamf Pro 要匯入前必須先把憑證跟私鑰合併成一個pfx檔案才可以匯入  
所以要再次使用 OpenSSL 來操作  
輸入以下的指令就可以囉  

``openssl pkcs12 -export -in server.crt -inkey server.key -out server.pfx``

在輸入之後會需要填寫一組密碼，等等會用到  

▼ 打開 jamf Pro 管理介面  
▼ 點選 Computers  
▼ 點選 Configuration Profiles  
▼ 點選 New  
▼ 在 NAME 填寫描述檔名稱  
▼ 在 distribution method 選擇 install automatically  
▼ 點選 network  
▼ 點選 configure  
▼ 在 SERVICE SET IDENTIFIER 填寫 Wi-Fi 的名稱  
▼ 在 SECURITY TYPE 點選 WPA2 Enterprise  
▼ 在 Protocols 點選 PEAP  
▼ 在 USERNAME以及 PASSWORD 填寫連線的帳號密碼，在 VERIFY PASSWORD 再填一次密碼  
▼ 點選 Certificate  
▼ 點選 configure  
▼ 在 CERTIFICATE NAME 填寫憑證名稱  
▼ 在 SELECT CERTIFICATE OPTION 選擇 Upload  
▼ 點選 Upload Certificate 來上傳剛才合併的憑證  
▼ 在 PASSWORD 以及 VERIFY PASSWORD 填寫剛才合併憑證的密碼  
▼ 點選 ＋ 新增憑證  
▼ 在 CERTIFICATE NAME 填寫憑證名稱  
▼ 在 SELECT CERTIFICATE OPTION 選擇 Upload  
▼ 點選 Upload Certificate 來上傳 CA 憑證  
▼ 在 PASSWORD 以及 VERIFY PASSWORD 填寫 CA 憑證的密碼  
▼ 回到 network      
▼ 點選 trust  
▼ 點選 CA 憑證    
▼ 點選 Scope  
▼ 選擇要佈署的裝置  
▼ 設定完成後點選 Save  

### 疑難排解

Ｑ：為什麼 Wi-Fi 沒有自動連上?  
Ａ：先確定在系統偏好設定裡面的描述檔有 802.1x 網路描述檔，裡面必須要包含兩個憑證以及一個 Wi-Fi network  
      如果沒有出現的話請打開 terminal ，輸入下列指令  
      
``sudo Jamf recon``

再重新進入描述檔裡面查看有沒有出現802.1x 網路描述檔。  
