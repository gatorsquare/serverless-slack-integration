#簡介
使用 AWS 的 Lambda 及 API Gateway，以 Severless 的方式在 Slack 頻道中建立 slash command。

##需要了解的東西：
* **Slack Slash Commands**   https://api.slack.com/slash-commands
* **AWS 基本操作**（設定 AWS 帳號......等等）
  - **AWS Lambda function** https://aws.amazon.com/lambda
  - **AWS API Gateway** http://aws.amazon.com/api-gateway
* **Node.js** 略懂即可

#程式流程架構
1. 使用者在 Slack 頻道中輸入 slash command，其形式為 `/指令` 指令字串。

2. Slack 會以 HTTP POST 向外部 url 發出一堆 content-type header 設定是 `application/x-www-form-urlencoded` 的資料。

3. 使用 API Gateway 接收這些資料，轉換成 JSON 格式，再傳給 AWS Lambda。   

4. AWS Lambda 執行運算、處理收到 JSON 資料，做條件判斷。視指令性質，可以將資料存到外部資料庫、呼叫外部的 API 等等，最後再將結果轉換成 Slack 可以接受的格式。

5. 資料先回傳給 API Gateway，再回傳 Slack 顯示。

#說明與操作步驟
* **AWS Lambda**
  - 選擇 Create a Lambda Function 新建一個 lambda function。
  
  - 選擇 slack-echo-command 的 blueprint。（若使用 python，可選擇 slack-echo-command-python）
  
  - 設定 lambda function 的名稱，填寫相關資料，Runtime 選擇為 Node.js（預設）
  
  - 往下拉可以看到 Edit Code Inline 的畫面有一大串可怕的 code，不過一大半都是解釋步驟，如果沒有加密需求的話，可以全部註解掉，並貼上以下簡單的指令來做測試。
  
  ```
  exports.handler = function(event, context) {
  context.succeed("Hello World!");
};
  ```
  
  - 拉到下方，Role 選擇 lambda_basic_execution，這時會跳出另一個視窗，按下 Allow 即可。
   
  - 其他欄位保留系統預設值，然後按下頁面右下角的 Next。

  - 進入 configure endpoint 的頁面，做如下設定：
    - API endpoint type：API Gateway
    - API name：自訂
    - Resource name：會出現之前取的 function 名稱，無須更動
    - Method：POST
    - Deployment stage：自訂
    - Security：Open  （一定要改成 Open，不然無法使用）
    - 設定完後按 Next 進入下一步。
    
  - 進到 review 畫面，檢視一下所有的設定是否正確，按 Create Function。
  
  - 接著畫面會跳到 API endpoints，那串 url，就是之後設定 slack command 時需要填入的 url。 
  
  - 注意事項：這時如果點上面的 code，跳回程式碼的頁面，會發現有些東西變成亂碼，內心可能會小小罵一下髒話，但不用擔心，再改回來就好了。

  - 在進入其他部分前，我們先來測試 lambda function 可否運作。按下左上方藍色的 save and test。然後再跳出的視窗中按下 save and test。這時會看到頁面下方小小的跑了一下，出現了執行成功的訊息，下方顯示 Hello world。

  - 這時如果回到 API endpoints 的地方，手賤點一下 url，會出現
  ```
  {"message":"Missing Authentication Token"}
  ```
  此時內心會發出非常多的OS，不過這只是表示我們該做的還沒做完，還要去設定 API Gateway。
  
* **API Gateway**
  - 進到 API Gateway 的 console 中，找到剛剛自訂的 API name 的方塊，點進去。

  - 這時畫面中會出現兩欄，左欄為 Resources，右欄則為 / Methods。在左欄中，如果之前設定正確的話，會出現之前設定的名稱，其下方會出現 POST。

  - 點 POST，右欄會出現流程圖，確認最右方的 lambda 的名稱為自訂的名字，若正確的話，點 lambda 名稱會連回 lambda 設定頁面。

  - 點選 Integration Request，進入後，integration type 選擇 Lambda Function

  - 點開 Mapping templates，點選 add mapping template，在 content-type 處，將預設的 application/json 改成 application/x-www-form-urlencoded，按下旁邊的勾。

  - 再按下最右邊的鉛筆圖示，將 Input passthrough 改成 mapping template，然後在下面跳出來的區塊中，輸入
  
    `{ "body": "$input.path('$')" }` ，再按上方的小勾。

  - 點選左上方的 Deploy，選擇之前設定的 deploy stage，確定。

  - 這時如果點選項左欄的 POST，應會在右欄上方看到一串 invoke url，這個 url 應該要和你在 lambda function endpoint 中那頁的 url 一樣。

  - 測試：點左上角的 Amazon API Gateway，選擇剛剛設定的 API 方塊，點左欄的 POST，右欄出現流程圖後，點一下最左邊圖塊的 TEST，進入測試畫面，按下右方藍色的 Test，如果前面全部設定正確的話，會出現 Hello World。
  
* 設定一堆以後，故事還沒結束，落落長的設定還要繼續。上面的設定只是確定 Lambda 和 API Gateway 有串連好而已，這時可以去上個廁所吃個點心喝個飲料休息片刻再回來。

* **建立 index.js**
  - 這部分請在本地端建立一個可以執行的檔案。範本程式可參考（連結待放）

  - 將 index.js 及其相關 dependencies 全部壓縮成 自訂名稱.zip。

  - 回到 lambda 的 code 界面，選擇上傳 Upload a .ZIP file，上傳後記得按下 Save 儲存。
  
* **Slack （先確定要有權限）**
  - 連到 https://團隊名稱.slack.com/apps/build/custom-integration
  
  - 選擇 slash command
  
  - 輸入指令名稱，並按下綠色 Add Slash Command Integration
 
  - 在 URL 欄位中填入 lambda function endpoint 那個 url。
  
  - 按下 Save Integration。
  
* 正常來說，接下來就可以在 Slack 頻道中使用新的指令了。（OS：但是人生總是有那麼多不正常的狀況，所以 good luck on coding and debugging!）  
  
