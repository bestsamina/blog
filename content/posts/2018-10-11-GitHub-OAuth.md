+++
title = "申請 GitHub OAuth 權限紀錄"
tags = ["GitHub", "OAuth", "TGmeetup", "RFC", "Authorization"]
categories = ["Network", "Authorization"]
date = 2018-10-11T01:10:15+08:00
+++

## 前言
會來使用 GitHub 的 [OAuth](https://zh.wikipedia.org/wiki/%E5%BC%80%E6%94%BE%E6%8E%88%E6%9D%83) 是因為發現 [TGmeetupBot](https://github.com/TGmeetupBot) 在開 issue 的時候，總是會開到重複的 Event Issue。  
在經過一番的拆解後，發現雖然都是使用 Basic authentication ，用 Postman 是可以看到所有的 Issue ，但是用程式卻是只能看到最新的 30 筆 issue。  
所以就需要 query 更多的 GitHub API 來比對沒撈到的 issue 用搜尋的方式確保不會開到重複的 Issue。  
但卻碰上 [Rate limiting](https://developer.github.com/v3/#rate-limiting) 的問題了。  
因為每小時只能有 60 個請求，但現在卻超過了。  
所以就要來用 OAuth 了。  
  
以下是參考 [Authorizing OAuth Apps](https://developer.github.com/apps/building-oauth-apps/authorizing-oauth-apps) 申請 OAuth 的紀錄。(讓自己下次在申請 OAuth 時，可以翻閱這個筆記，快速記得。)  

## 1. 申請 OAuth2 認證

到 https://github.com/settings/applications/new 註冊成為 OAuth 使用者。  

- 必填欄位 "Application name" "Homepage URL" 與 "Authorization callback URL"
    - Application name：為 Application 名字的字串 (這邊可以隨意的字串，方便管理即可。)
    - Authorization callback URL：請填入 http://localhost:8000/ (這欄位是為了拿一次性的 access token，後面會提供如何設定的範例，這會需要在本機上進行。)
    - Homepage URL: 隨意的 URL 
  
範例圖  
![](https://i.imgur.com/gIxMjSA.png)  


## 2. 取得 Client ID 和 Client Secret

完成步驟一後，會獲得 ID 和 secret 的值。  

![](https://i.imgur.com/5iYu4TO.png)

## 3. 在本機端啟動 SimpleHTTPServer

在拿 access token 之前，我們先來在本機端開個 8000 port。  

- 安裝 python2 環境在本機上。
- 在你的任意目錄底下輸入 `python -m SimpleHTTPServer 8000`

## 4. 取得 access token

於瀏覽器的網址列中輸入以下連結，並將 client_id 等於後的值填上 Client ID 的 key 值  

```
https://github.com/login/oauth/authorize?client_id=ce16xxxxxxxxxxxxx
```

## 5. 同意該 OAuth 使用者可以使用你的帳號

接著會跳出網頁請你點選確認選項，請點選 "Authorize GitHub_user"。  
![](https://i.imgur.com/5UvBEfu.png)  

## 6. 轉導向到自己本機的 8000 port 並取得 access token

如圖： 8939.....  
![](https://i.imgur.com/aQeoOsz.png)  

## 7. 來拿 access token

以下提供兩種方法來取得 refresh token。  
方法1: 使用 terminal 或是程式撰寫來取得  
方法2: 使用 Postman 來取得  

#### 方法 1: terminal 或是程式撰寫

此方法僅用描述的方式，以下為一些注意項目  

1. 使用 POST 的方法取得
2. Header encoded 方法使用 `application/x-www-form-urlencoded`
3. URL 為 `https://github.com/login/oauth/access_token`
4. body 中的 request key 與 value 如下

```
client_id=CLIENT_ID
&client_secret=CLIENT_SECRET
&code=CODE_YOU_RECEIVED_FROM_THE_AUTHORIZATION_RESPONSE
```

- CLIENT_ID: 在 GitHub OAuth 中拿到 client id 的 key 值
- CLIENT_SECRET: 在 GitHub OAuth 中拿到的 secret 值
- CODE_YOU_RECEIVED_FROM_THE_AUTHORIZATION_RESPONSE: 為上一步驟獲得的 code 值

#### 方法 2: 使用 Postman

Postman 是一個 API 的測試工具，您也可以使用其他類似的工具進行，以下為 Postman 的示範  

- 以下步驟請搭配本張圖示進行

![](https://i.imgur.com/PqQxFaE.png)

1. 開起 Postman 
2. 在右上選擇 POST 方法
3. 在 Enter request URL 輸入 `https://github.com/login/oauth/access_token`
4. 點選 Body
5. 點選 `application/x-www-form-urlencoded`
6. 輸入 key 和 value

| Key | Value |
| --- | --- |
| client_id | 在 GitHub OAuth 中拿到 client id 的 key 值 | 
| client_secret | 在 GitHub OAuth 中拿到的 secret 值 |
| code | 在上一步驟獲得的 code 值 |

7. 點選 **Send** 取得 access token

  
- Ref:
    - https://hackmd.io/s/ByrxYmi4G#  
    - https://blog.yorkxin.org/2013/09/30/oauth2-1-introduction.html
