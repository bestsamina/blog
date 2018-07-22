+++
title = "[TGmeetup] 之詳細安裝操作步驟"
tags = ["open-source", "TGmeetup", "meetup"]
categories = ["TGmeetup"]
date = 2018-01-28T14:51:05+08:00
+++

此篇文章內容同步於 [https://hackmd.io/s/ByrxYmi4G](https://hackmd.io/s/ByrxYmi4G) 。

# 安裝流程

- Step 1. 申請 meetup_api 的 auth Key
   1. 申請 OAuth2 認證
   2. 取得 Key 和 secret
   3. 在本機端啟動 SimpleHTTPServer
   4. 取得 access token
   5. 同意該 OAuth 使用者可以使用你的帳號。
   6. 轉導向到自己本機的 8000 port 並取得一次性的 access token
   7. 準備取得 refresh token
   8. 取得 refresh token 與 access token
- Step 2. 下載專案並編輯 API.cfg 檔
- Step 3. 進行安裝

# 安裝說明

請依據以下安裝說明進行，謝謝。  

# Step 1. 申請 meetup_api 的 auth Key

以下操作步驟皆是參考[官方的說明文件](https://www.meetup.com/meetup_api/auth/#oauth2)來進行，並提供簡單的取得方法。  

### 1. 申請 OAuth2 認證

- 到 https://secure.meetup.com/meetup_api/oauth_consumers/create/ 註冊成為 OAuth 使用者。
- 必填欄位 "Consumer name" 與 "Redirect URI"。
    - **Consumer name**：為使用者名字的字串 (這邊可以隨意的字串，方便您管理即可。)
    - **Redirect URI**：請填入 `http://localhost:8000/` (這欄位是為了拿一次性的 access token，後面會提供如何設定的範例，請在本機上進行。)
- 範例圖
![](https://i.imgur.com/vV6xszF.png)  

### 2. 取得 Key 和 secret

完成步驟一後，會獲得 key 和 secret 的值。  
![](https://i.imgur.com/6ML2bJ1.png)


### 3. 在本機端啟動 SimpleHTTPServer 

- 在拿 access token 之前，我們先來在本機端開個 8000 port。
    1. 安裝 python2 環境在本機上。
    2. 在你的任意目錄底下輸入 `python -m SimpleHTTPServer 8000`

### 4. 取得 access token

- 於瀏覽器的網址列中輸入以下連結，並將 YOUR_CONSUMER_KEY 的字串取代成你在 Meetup OAuth 所獲的的 key 值

```
https://secure.meetup.com/oauth2/authorize
    ?client_id=YOUR_CONSUMER_KEY
    &response_type=code
    &redirect_uri=http://localhost:8000/
```

- 如本文範例需要在網址列輸入

```
https://secure.meetup.com/oauth2/authorize
    ?client_id=7rt32r...........
    &response_type=code
    &redirect_uri=http://localhost:8000/
```

### 5. 同意該 OAuth 使用者可以使用你的帳號。
接著會跳出網頁請你點選確認選項，請點選 "Allow"。  
![](https://i.imgur.com/9w0OyKE.png)  

### 6. 轉導向到自己本機的 8000 port 並取得一次性的 access  token

![](https://i.imgur.com/jr2gc6J.png)  

### 7. 準備取得 refresh token
以下提供兩種方法來取得 refresh token。  
方法1: 使用 terminal 或是程式撰寫來取得  
方法2: 使用 Postman 來取得  

#### 方法 1: terminal 或是程式撰寫

此方法僅用描述的方式，以下為一些注意項目  

1. 使用 POST 的方法取得
2. Header encoded 方法使用 `application/x-www-form-urlencoded`
3. URL 為 `https://secure.meetup.com/oauth2/access`
4. body 中的 request key 與 value 如下

```
client_id=YOUR_CONSUMER_KEY
&client_secret=YOUR_CONSUMER_SECRET
&grant_type=authorization_code
&redirect_uri=SAME_REDIRECT_URI_USED_FOR_PREVIOUS_STEP
&code=CODE_YOU_RECEIVED_FROM_THE_AUTHORIZATION_RESPONSE
```

- YOUR_CONSUMER_KEY: 為您在 Meetup OAuth 中拿到的 key 值
- YOUR_CONSUMER_SECRET: 為您在 Meetup OAuth 中拿到的 secret 值
- SAME_REDIRECT_URI_USED_FOR_PREVIOUS_STEP: 為 `http://localhost:8000/`
- CODE_YOU_RECEIVED_FROM_THE_AUTHORIZATION_RESPONSE: 為您在上一步驟獲得的 code 值

#### 方法 2: 使用 Postman

Postman 是一個 API 的測試工具，您也可以使用其他類似的工具進行，以下為 Postman 的示範  
- 以下步驟請搭配本張圖示進行  
![](https://i.imgur.com/fY0TWXj.png)

1. 開起 Postman 
2. 在右上選擇 POST 方法
3. 在 Enter request URL 輸入 `https://secure.meetup.com/oauth2/access`
4. 點選 Body
5. 點選 `application/x-www-form-urlencoded`
6. 輸入 key 和 value

| Key | Value |
| --- | --- |
| client_id | 您在 Meetup OAuth 中拿到的 key 值 | 
| client_secret | 您在 Meetup OAuth 中拿到的 secret 值|
| grant_type | authorization_code|
| redirect_uri | `http://localhost:8000/`|
| code | 為您在上一步驟獲得的 code 值 |

7. 點選 **Send**

### 8. 取得 refresh token 與 access token

refresh token: 因為 access token 有時限性，所以當超過時限時，皆需要使用 refresh token 重新拿 access token。
  
```
{
    "access_token": "5b99e.........................a1c56",
    "refresh_token": "461cb.....................0b1e0",
    "token_type": "bearer",
    "expires_in": 3600
}
```
  
以上第一階段取得 Meetup OAuth 的 key 就完成囉！  

# Step 2. 下載專案並編輯 API.cfg 檔

1. git clone 整個專案

```
$ git clone https://github.com/TGmeetup/TGmeetup.git
```

2. 切換到 TGmeetup 的資料夾

```
$ cd TGmeetup
```

3. 複製 API.cfg.sample 檔案到 API.cfg

```
$ cp API.cfg.sample API.cfg
```

4. 編輯 API.cfg

```
[MEETUP_API]
API_URL = https://api.meetup.com/
API_KEY = 7rt32........
client_secret = apafji................68hso
refresh_token = edb54a1f5d...............eb92
```

- API_KEY: 為您在 Meetup OAuth 中拿到的 key 值
- client_secret: 為您在 Meetup OAuth 中拿到的 secret 值
- refresh_token: 為您在 [Step1 中的第八步驟](#8-%E5%8F%96%E5%BE%97-refresh-token-%E8%88%87-access-token)取得的 refresh_token 值

# Step 3. 進行安裝

```
$ sh install.sh
```

---

完成以上就可以開始來使用 `tgmeetup` 的指令囉！ 想知道更多，請到 https://github.com/TGmeetup/TGmeetup 。:)

### 相關系列文：
[[TGmeetup] 就讓這專案蒐集世界各地的技術社群資訊吧！](/series/tgmeetup/)

