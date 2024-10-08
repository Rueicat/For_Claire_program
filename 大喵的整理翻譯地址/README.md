# 大喵的整理資料翻譯地址

因為大喵要接收來自各家公司的資料, 對方有對方整理資料的邏輯, 大喵轉交出去有需要的邏輯與排版, 所以做了這個小工具

## WorkFlow

1.選擇指定資料夾(內含需要處理的excel檔案)

2.依照需要分類的項目, 寫入新的`result.xlsx`, 設置需求項目在row 1 (proproties)

3.要邏輯判斷沒有按照規定填寫資料的客戶, 需要排除並做人工確認

4.所以輸出檔案sheet[0]放處理好的檔案; sheet[1]放被排除的檔案, 需要人工確認

> 為了方便查閱, 有設定echo 顯示錯誤的檔名, sheet名稱.

## Microsoft Azure Translator API 建置

這個弄很久...因為微軟的文件看了老半天, chatgpt問了老半天都找不出問題, 一直出現`400` 請求錯誤
```powershell
翻譯 API 呼叫失敗: 遠端伺服器傳回一個錯誤: (400) 不正確的要求。
錯誤內容: {"error":{"code":400074,"message":"The body of the request is not valid JSON."}}
```

後來發現是微軟要求JSON格式是**一維陣列**, 恩, 看起來好像沒什麼問題, 我們程式在建置Json時, 不就是一維陣列嗎?

但是, 因為我是抓這個API來一個一個翻譯對應的excel格子中的文字, 查資料了老半天才知道, <span style="color: red;">在powershell中, 如果陣列中只有一個元素時, 他會自己把它變成Object(對象, 物件), 不是Array(陣列, 數組)</span>, 另外, 他也屬於巢狀結構式, 預設只會分析到第二層, 所以powershell 中有提供`-Depth N`參數, 不然超過的都會變成String

>NOTE: 如何解決上述問題? (一個元素會被轉成Object, 而不是Array 的問題)

- 方法一：自己定義函數強制序列化為Array
```powershell
function ConvertTo-JsonForceArray($InputObject, $Depth) {
    $json = $InputObject | ConvertTo-Json -Depth $Depth
    if ($InputObject.Count -eq 1 -and $json.StartsWith('{')) {
        return "[$json]"
    } else {
        return $json
    }
}
```
- 方法二：用`compress`參數壓縮成單行後, 自己加陣列框框[ ]

```powershell
$bodyJson = $body | ConvertTo-Json -Depth 2 -Compress
if ($body.Count -eq 1) {
    $bodyJson = "[$bodyJson]"
}
```

- 方法三：升級powershell 版本

因為一般電腦裝的都是PS 5.1版, 現在的powershell 7 版本, 有多一個參數`-AsArray`強制轉換


>NOTE：以上程式範例

 #### 物件 = {}, 陣列 = [], 一個是大括號, 一個是中括號

 ```powershell
 $body = @(
    @{
        'title_one' = "123"
        'title_two' = "999"
    }
) | ConvertTo-Json -Compress
 ```
這樣得到的結果是:\
{"title_one":"123","title_two":"999"}

`這是Object`(大括號)

當有兩個元素(?)在裡面時...

```powershell
$body = @(
    @{
        'title_one' = "123"
        'title_two' = "999"
    }

    @{
        '這是第二個元素' = "5555555"
    }

) | ConvertTo-Json -Compress
```

得到的結果是: \
[{"title_one":"123","title_two":"999"},{"這是第二個元素":"5555555"}]

`這是Array`(有中括號)


也許我很菜的關係, 第一次知道這樣的分別, 設置這個微軟的翻譯API...我弄了好久...特別在此做筆記, 下次不要忘了

## 處理編碼問題

之前都在自己的環境玩耍打code, 或是在人家建置好的環境裡(例如google apps script)做API連結, 後來在跨環境與跨平台時, 會慢慢發現有編碼的問題要處理, 尤其我們是中文系統的使用環境, 在處理文字訊息時, 對於一般人家開源提供的code 或要建置API連結, 可能都需要轉碼編制(UTF-8)

這裡也不例外, 我使用的powershell 基本上是UTF16 BE 的編碼, 但微軟 translator API 是需要接收UTF-8的編碼, 所以要在程式裡面做轉換

```powershell
 $EncodingBody = New-Object System.Text.UTF8Encoding $false
            
            $bodyJson = ConvertTo-JsonForceArray -InputObject $body -Depth 2
            $newbody = $EncodingBody.GetBytes($bodyJson)
```

不要小看這一行`[System.Text.Encdong]`的功能, 我花了很多時間才對編碼比較有認識和體會...

## 效能問題

最後是有發現效能的問題, 我是一筆一筆上API請求, 這樣其實效率很慢, 程式要跑很久(目前大喵給我的測試檔約1500筆資料, 要花3分鐘)

當然這程式不只是單純翻譯而已, 還包含依照寫好的邏輯去整理每一個excel資料, 彙整成一個報表. 不過想換研究其他東西了, 效能的部分等之後回頭來再說吧