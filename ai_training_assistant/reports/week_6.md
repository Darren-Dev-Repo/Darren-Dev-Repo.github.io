# 2026/03/31 Week 6 進度回報：訓練者訓練日誌結構與 Prompt 設計
> 🤖 與 Gemini Pro 的對話記錄連結: [🔗](https://docs.google.com/document/d/1fA09hwy9DPun9yHdjntHFbuQ6Y91ccPPFcA10Q4pae0/edit?tab=t.0)   
> 🏋️ Project Repository: [🔗](https://github.com/Darren-Dev-Repo/AI_Training_Assistant)   
<- [Week 5](https://docs.google.com/document/d/1Dj_fMQAJMHkxeaG_5NsKxR5JSTiESkBiNHwY51bTroY/edit?tab=t.0)   

## 【給老師的回覆】關於 RAG 架構的近期演進與本專案的應用展望
### RAG 簡介與演進概況
RAG (Retrieval Augumented Generation) 是一項將自然語言處理應用於查詢的方式，其目的是讓 AI 只根據限定的資料回應而非憑空杜撰，降低幻覺機率。   
傳統的 RAG 根據原本的資料庫，建立一個向量資料庫，查詢時會從中尋找與自然語言輸入中字詞向量相似度最高的結果。這項技術也被稱為「語意檢索」。   
如今技術層面的突破使得 RAG 技術不再侷限於文字向量檢索，以下簡單說明一些典型的實作方法：
* Graph RAG: 將不同的資料來源以網狀圖連接，強化 AI 對於知識間的關聯。
* Adaptive RAG: 系統根據問題難度自動決定檢索路徑。
* Long Context: 暴力破解法。目前的 LLM 能一次接收數百個 Token 含量的文本，故可以直接將需要查詢的資料寫入 Prompt，而不須切分字詞。

### RAG 實際專案經驗 - [🏋️-Fitness-RAG-Agent-Pipeline](https://github.com/Darren-Dev-Repo/-Fitness-RAG-Agent-Pipeline/)
除了期初的 Coze 工作流外，2026年3月中下旬，我以ChromaDB實作傳統的語意檢索，根據使用者的文字需求找出適合的健身動作。   
這份專案讓我了解原始 RAG 的完整流程，對於接下來 AI Training Assistant App 的開發所需的進階 RAG 技術，提供紮實基礎。

### 本專案將採用的 RAG 架構
ChromaDB 專案達成了語意檢索的功能，但單純的向量檢索無法進行精確的數學運算 (例如課表的重量遞增等進步追蹤指標)，也無法處理帶有時間序列的商業邏輯。因此本專案將採用有狀態的 Hybrid RAG 與 Agentic Workflow 取代傳統靜態檢索：

1. LLM 意圖解析與強型別約束：
本週我將重心放在資料庫 Schema 的設計。有別於傳統 RAG 輸出模糊的自然語言，本專案將 LLM 定位為系統的意圖解析器，從。
透過防禦性 Prompt，強制 LLM 讀取訓練者的真實訓練回饋後，正確輸出為後端程式可運算的 Workout_Log.json 結構。

2. 結構化個人狀態檢索：
未來的決策引擎在推論前，會先檢索儲存於資料庫中該名訓練者的 Current_State.json (包含各動作當前重量、卡關次數)，讓 LLM 擁有結構化的長期記憶 (狀態)，而非每次對話都從零開始。

3. 動態領域知識注入：
為解決 Prompt 耦合與幻覺問題，未來將捨棄在 Prompt 中寫死動作名稱的做法。系統會在執行期間先從資料庫撈出合法的動作清單，動態注入到 Prompt 中，約束 LLM 的生成邊界。

4. 狀態凍結與決策引擎：
當 RAG 檢索出替代動作並完成記錄後，系統將啟動狀態凍結演算法，保護課表的主項目的進度不被替代動作的資料影響，結合 AI 的變動性與軟體工程的確定性。

## GitHub Page 建立與設計
本週起，學習進度報告將會公布於我的 GitHub Page 上，提升老師與同學的易讀性。  
   
我先參考 GitHub Page 官方文件的 [Quickstart](https://docs.github.com/en/pages/quickstart) 快速建立一個基礎的頁面。  
後續於本地規劃下列檔案結構，並推送至雲端倉庫，明確制定資料間的層級，以利未來擴充。

```Plaintext
Darren-Dev-Repo.github.io/
├── ai_training_assistant/
│   ├── reports/
│   │   └── week_6.md
│   └── README.md
├── _config.yml
└── README.md
```

## JSON
JSON (Javascript Object Notion) 被應用於應用程式前後端間的資料傳輸與控制，利用 LLM 能輸出 JSON Format 的特性，讓 LLM 從訓練者的自然語言輸入中辨識並擷取出可以被儲存的資料，這些資料將可以藉由邏輯與運算的模組處理以得到有價值的資訊，或是傳遞給決策模組以改善訓練建議。   
JSON裡的每一個物件皆由 "key": value 組成，value 具有許多資料型態，這些資料型態能和 Python 等其他介面互通。   
物件的集合可以由 [] (陣列) 或是 {} (物件) 表示。兩者的差異在於資料結構的特性，[] 中的資料是連續儲存的，而 {} 的則是對映的，根據所需情況選用不同的資料結構。

## 訓練日誌範例設計
訓練日誌分成 Workout_Log.json 和 Current_State.json 兩種，Workout_Log.json 記錄每一次的訓練前、中、後的情況，而 Current_State.json 記錄訓練者當前的最新狀態。      
我先設計一個情境如下：
> 2026年4月5日，訓練者這次計畫上的動作是槓鈴深蹲、槓鈴臥推和槓鈴划船5x5。這時健身房充斥著下班人潮，好險深蹲架還有空位，不過訓練者不久前滑雪弄傷了肩膀，而且晚點還要參加其他課程，今天只有40分鐘可以訓練。
> 於是，他決定停練臥推一次，將槓鈴深蹲改為3組5下的腿推，並降低槓鈴划船的重量。
<details>
<summary>接著按照情境，嘗試先自行撰寫 Workout_Log.json</summary>
```json
{
    "_comment": "This format records information of each workout of the trainer.",

    "date": "2026-04-05",
    "invalid_eqipments": ["dumbbells", "cable_machine", "trapbar"],
    "exhausted_or_injured_body_parts": ["shoulder"],
    "avilable_time_in_minutes": 40,

    "performed_exercises": [
        {
            "exercise_name": "leg_press",
            "is_exercise_in_program": false,
            "performed_sets": [
                {
                    "set_no.": 1,
                    "weight_in_kilograms": 70,
                    "performed_reps": 5,
                    "RPE": 7,
                    "rest_time_in_seconds": 150,
                    "is_last_set": false
                }, {
                    "set_no.": 2,
                    "weight_in_kilograms": 70,
                    "performed_reps": 5,
                    "RPE": 8,
                    "rest_time_in_seconds": 150,
                    "is_last_set": false
                }, {
                    "set_no.": 3,
                    "weight_in_kilograms": 70,
                    "performed_reps": 5,
                    "RPE": 10,
                    "rest_time_in_seconds": "N/A",
                    "is_last_set": true                    
                }
            ]
        }, {
            "exercise_name": "barbell_row",
            "is_exercise_in_program": true,
            "performed_sets": [
                {
                    "set_no.": 1,
                    "weight_in_kilograms": 20,
                    "performed_reps": 12,
                    "RPE": 6,
                    "rest_time_in_seconds": 90,
                    "is_last_set": false
                }, {
                    "set_no.": 2,
                    "weight_in_kilograms": 20,
                    "performed_reps": 12,
                    "RPE": 7,
                    "rest_time_in_seconds": 90,
                    "is_last_set": false
                }, {
                    "set_no.": 3,
                    "weight_in_kilograms": 20,
                    "performed_reps": 12,
                    "RPE": 7,
                    "rest_time_in_seconds": 90,
                    "is_last_set": false                    
                }, {
                    "set_no.": 4,
                    "weight_in_kilograms": 20,
                    "performed_reps": 12,
                    "RPE": 7,
                    "rest_time_in_seconds": 90,
                    "is_last_set": false
                }, {
                    "set_no.": 5,
                    "weight_in_kilograms": 20,
                    "performed_reps": 12,
                    "RPE": 7,
                    "rest_time_in_seconds": "N/A",
                    "is_last_set": true                    
                }
            ]            
        }
    ]
}
</details>

實作初版後，為了確保後端程式解析的安全與邏輯完整性，我進行了以下架構微調：
1. 資料型態一致性：將初版字串 "N/A" 改為 null，避免後端語言解析 Integer 時引發例外錯誤。
2. 建立替代關聯：新增了 substituted_for 欄位與 skipped_exercises 陣列。這確保了系統能辨識*腿推*是為了替代*深蹲*，而非憑空出現的獨立動作。
<details>
<summary>這是第二版的 Workout_Log.json</summary>
```json
{
    "_comment": "This format records information of each workout of the trainer.",

    "log_id": "L005",
    "date": "2026-04-05",
    "pre_workout_status": {
        "pre_workout_fatigue_level": 8,
        "unavailable_equipment": ["dumbbells", "cable_machine", "trapbar"],
        "exhausted_or_injured_body_parts": ["shoulder"],
        "available_time_in_minutes": 40,
        "skipped_exercise": ["bench_press"]
    },

    "performed_exercises": [
        {
            "exercise_name": "leg_press",
            "is_exercise_in_program": false,
            "substitute_for": "barbell_squat",
            "performed_sets": [
                {
                    "set_no.": 1,
                    "weight_in_kilograms": 70.0,
                    "performed_reps": 5,
                    "RPE": 7,
                    "rest_time_in_seconds": 150,
                    "is_last_set": false
                }, {
                    "set_no.": 2,
                    "weight_in_kilograms": 70.0,
                    "performed_reps": 5,
                    "RPE": 8,
                    "rest_time_in_seconds": 150,
                    "is_last_set": false
                }, {
                    "set_no.": 3,
                    "weight_in_kilograms": 70.0,
                    "performed_reps": 5,
                    "RPE": 10,
                    "rest_time_in_seconds": null,
                    "is_last_set": true                    
                }
            ]
        }, {
            "exercise_name": "barbell_row",
            "is_exercise_in_program": true,
            "substitute_for": null,
            "performed_sets": [
                {
                    "set_no.": 1,
                    "weight_in_kilograms": 20.0,
                    "performed_reps": 12,
                    "RPE": 6,
                    "rest_time_in_seconds": 90,
                    "is_last_set": false
                }, {
                    "set_no.": 2,
                    "weight_in_kilograms": 20.0,
                    "performed_reps": 12,
                    "RPE": 7,
                    "rest_time_in_seconds": 90,
                    "is_last_set": false
                }, {
                    "set_no.": 3,
                    "weight_in_kilograms": 20.0,
                    "performed_reps": 12,
                    "RPE": 7,
                    "rest_time_in_seconds": 90,
                    "is_last_set": false                    
                }, {
                    "set_no.": 4,
                    "weight_in_kilograms": 20.0,
                    "performed_reps": 12,
                    "RPE": 7,
                    "rest_time_in_seconds": 90,
                    "is_last_set": false
                }, {
                    "set_no.": 5,
                    "weight_in_kilograms": 20.0,
                    "performed_reps": 12,
                    "RPE": 7,
                    "rest_time_in_seconds": null,
                    "is_last_set": true                    
                }
            ]            
        }
    ],

    "post_workout_feedback": "Feel pumped!"
}
</details>
訓練的課表實務上容易因為臨時更換器材而導致進度計算混亂。藉由上述設計的 substituted_for 欄位，未來的運算模組將能執行狀態凍結，即當訓練者因器材不足或受傷而執行替代動作時，系統能如實記錄當日的訓練容量，但同時凍結課表中原運動項目的進度，不對課表的資料造成影響。   

針對缺漏的部分修正並補上後，隨後寫完成了 Current_State.json 的設計。

## Prompt 設計
之前利用 Coze 的工作流實作過讓 LLM 輸出資料並儲存至資料庫的流程，這次要做的本質上是一樣的，但是可於文字邏輯處理以外的模組 (例如數學運算) 運用。   
AI 建議我使用 Zero - shot 將 Workout_Log.json 的格式寫死在 Prompt 中，我暫時採用這個做法以利快速驗證，但之後會修改避免耦合問題。   
為了確保 LLM 不會自作主張吐出錯誤的格式，我在 Prompt 中增加了以下防範性的限制：
1. 明確禁止 LLM 輸出 ```json 等 Markdown 標籤，確保後端的 json.loads() 不會因此引發 Decode Error。
2. 要求 LLM 即使遇到未提及的資訊，也必須輸出 null 或 []，嚴格禁止省略 Key，避免後端拋出 KeyError。

目前撰寫的 Prompt 經 GPT - 4o 處理可產生正確格式的文本，不過 LLM 目前無法正確推斷動作名稱與動作關係，且機制上，使用者沒有提及的資訊無法提取為資料。未來將從後端進行動態注入，讓 LLM 能從資料庫取得動作的資料。