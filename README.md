# 2024立委議題分析資料集｜報導者Data Team
2024.1.4

## 專案說明
- 本資料集由<a href="https://www.twreporter.org/">《報導者》</a>數據新聞小組整理，原始資料取自<a href="">立法院開放資料服務平台</a>，透過公民科技社群G0V工程師王向榮建置的<a href="">立院資料API</a>抓取而得。
- 本資料使用於報導者選舉專題<a href="https://www.twreporter.org/a/data-reporter-2024-election-10th-legislators-performance">【Data Reporter】你家立委4年來為哪些議題發聲？各黨立委質詢主題有何差異？馬上查閱每位立委5大關注議題</a>。

## 研究方法概述

### 一、資料蒐集與預處理
- 自立法院網站，整理第10屆立法委員名單資料，包含現職與離職立委共120位。自中選會選舉資料庫蒐集立委選區、性別、政黨資料，並自立法院官網整理個別立委曾參與的委員會清單。

- 自立法院開放資料服務平台抓取<a href="https://data.ly.gov.tw/getds.action?id=41">公報目錄</a>，並使用公民科技社群g0v工程師王向榮建置的<a href="https://ly.govapi.tw/">立院資料API</a>，以Python抓取本屆第1會期至第7會期的院會、委員會（含公聽會）、黨團協商、討論事項的會議紀錄html檔，讀取其中所有文字。

- 觀察公報中的發言紀錄規則，抓取個別立委的發言紀錄。例如：以「吳委員思瑤：」蒐集立委吳思瑤的所有發言，將該立委在同一次會議的所有發言內容整併為一筆。例外：一般來說，委員會會議主席為該會期召集委員，主席發言通常是請與會成員依序發言，與議題無顯著關聯，故略不計。另外，考量書面質詢占比較低，且質詢強度與口頭質詢相比較低，亦未納入分析。

- 扣除負責主持會議的院長游錫堃、副院長蔡其昌，總共蒐集到118人的<a href="https://github.com/data-reporter/10th_Legislator_Speech/blob/main/1_split_speeches.zip">23,599次發言</a>。《報導者》處理錯別字、亂碼問題後，將同一立委在同一會議中單次發言內容整併，共計23,511筆發言資料，欄位包含發言者、會議名稱、發言內容與原始公報連結。
- 將所有立委的基本資料及發言資料合併，依此進行分析。


### 二、機器學習與資料編碼
- 《報導者》數據團隊以R程式語言，透過中文斷詞套件Jieba對所有發言資料進行斷詞。
  
- Jieba套件的詞庫僅包含通用詞彙，因此我們以立委姓名、立法院動態資訊網法案追蹤平台彙整的<a href="https://lis.ly.gov.tw/billtpc/billtp">熱門議題</a>為基底，新增專業用語斷詞，檢查斷詞結果，來回調整斷詞清單及實際上無意義的停用詞，最後彙整出<a href="https://github.com/data-reporter/10th_Legislator_Speech/blob/main/2_special_words_zhTW.txt">1,130個專業用語</a>、<a href="https://github.com/data-reporter/10th_Legislator_Speech/blob/main/3_stopwords_zhTW.txt">4,783個停用詞清單</a>。
  
- 參考中研院人文社會科學研究中心博士後研究員何俊霆開源的<a href="https://github.com/justinchuntingho/Academia-Sinica-Topic-Modeling/blob/master/2_topicmodel.R">LDA主題模型（隱含狄利克雷分布）程式碼</a>，依照團隊需求進行數十次微調與修改，並使用在主題模型研究領域中常見的最佳主題數計算方法「<a href="https://doi.org/10.1016/j.neucom.2008.06.011">CaoJuan2009</a>」及「<a href="https://www.researchgate.net/publication/220895601_On_Finding_the_Natural_Number_of_Topics_with_Latent_Dirichlet_Allocation_Some_Observations">Arun2010>/a>」統計出最合適的主題數為200個。
  
- 模型根據所有資料分出200個主題後，會計算出每個主題的重點關鍵字，並根據每段發言內容，為其打上200個主題的分數（分數在0至1之間，愈高代表在該主題愈有代表性），此分數代表發言與特定主題的相關程度。例如：某發言與主題1的相關程度是0.53分、與主題2的相關程度是0.62分，我們會認為這段發言更多在討論主題2，藉此對大量文本進行快速分類與理解。


### 三、主題命名與特殊處理
我們初步以OpenAI開發的ChatGPT模型，根據每個主題最相關的前20大關鍵字、分數最高的前10個發言內容，辨識出主題名稱。例如：主題22的關鍵字是「CPTPP、談判、經貿、關稅協定、FTA」，就會將主題22命名為「國際經貿協議進度」。

初步命名後，《報導者》數據團隊根據前10大發言內容進行微調，經過與各領域專家討論、人工過濾及整併，最終確立<a href="https://github.com/data-reporter/10th_Legislator_Speech/blob/main/4_topics.csv">178個有效主題</a>、共22,786個發言，以下為特殊資料處理：

- 有3個主題分別是「購屋」、「租屋」及「建築規則更新」。「購屋」面向包含實坪制、囤房稅、房地合一稅等；「租屋」面向包含租屋黑市、租金補貼、租賃專法等；「建築規則更新」則包含容積率管制、公設比及危老建築更新等。由於這3個面向的相關發言都在「居住正義」的討論脈絡下，因此在發言人數、次數統計上將這3個主題合併為主題「居住正義」。
  
- 原有一主題與「預算刪減、凍結」相關，主題涵蓋各領域、各部會的預算審查，與專家討論後認為「預算審查」乃立委主要職務之一，與「議題探討」性質不同，因此將原被歸類在該主題的471筆發言重新計算，找出各發言分數第2高的主題。
  
- 「外交處境與進展」有拆分為兩主題，然兩主題的關鍵字、發言主軸極為相似，皆談及台灣的國際參與、實質外交情形，較難區分，因此合併計算。
  
- 有4個主題包含2個明顯不同的議題，因關鍵字組合部分重疊，被模型計為同一主題，分別是「姓名條例／中藥規範」、「災防預測／黃牛防範」、「18歲公民權／里鄰長事務費」、「數位身分證／國務機要費案」，我們觀察這4個主題的所有發言，將4個主題拆分為8個，其中有少數發言經人工判斷不屬於上述任何主題，則刪去不採用。


### 四、資料分析與研究限制

<b>資料分析</b>：完成主題命名、所有發言歸類後，《報導者》數據團隊依政黨、立委在各主題的發言數進行分析。立委議題儀表板部分陳列個別立委最常討論的議題，是以個別立委在178個議題中的發言數降冪排列，取前五高者。若有數個主題並列第五名，則以文本與主題的相關分數高者為優先。

例如：某立委的前六大議題發言數依序為10次、9次、8次、7次、6次、6次，議題A與議題B發言次數皆為6次，則觀察兩者的最高分文本分數（Gamma值），若議題A的最高分發言為0.6，議題B的最高分發言為0.7，則取議題B為第五大議題。

<b>研究限制</b>：
- 本研究是透過機器學習方式計算發言主題，並以最高分主題為依歸，然少數文本可能因有意義的關鍵字不足，在每個主題的分數都很低，即便採最高分主題，仍可能出現主題與發言內容不相符之情形。《報導者》數據團隊已確保立委表現儀表板中出現的摘要內容皆與主題相符，盡可能排除偏誤資料。

- 本報導以各主題發言次數作為指標，觀察立委對各大議題的討論度，然這不盡然代表議題在立法院的重要程度，或是特定政黨、個人對議題的關注程度。部分議題如18歲公民權、通姦除罪化可能因各黨皆有高度共識，因此討論度不如爭議議題。

＊感謝政治大學新聞系教授鄭宇君、政大傳播與資訊科學研究團隊「水火計畫」、政大傳播所楊聲輝、台北大學公共行政暨政策學系教授劉嘉薇協助給予研究方向；也感謝歐噴公司創辦人、g0v零時政府參與者王向榮提供立法院資料API。

## 聯絡我們
若對資料有任何建議或疑問，可寄信至報導者Data小組信箱：**data@twreporter.org**

*《報導者》是台灣第一個由公益基金會成立的網路媒體，秉持深度、開放、非營利的精神，致力於公共領域調查報導，與社會共同打造多元進步的媒體環境。*
