# 2026/04/07 Week 7 進度回報：設計後端流程暨下次訓練說明之系統提示詞
> 🤖 與 Gemini Pro 的對話記錄連結: [🔗](https://docs.google.com/document/d/1cULJvt3Ko2Q5VuhqNMzKcZNMeEUkcqtQt-kXjUtpWfk/edit?tab=t.0)   
> 🏋️ Project Repository: [🔗](https://github.com/Darren-Dev-Repo/AI_Training_Assistant)   
> <- [Week 6](week_6.md)  
> ______[Week 8](week_8.md) ->    

## 當前進度與本週目標
當訓練者以自然語言或圖形化介面輸入訓練紀錄，第一個 Agent 會從中提取這些資訊並輸出 [Workout_Log.json](https://github.com/Darren-Dev-Repo/AI_Training_Assistant/blob/main/Workout_Log.json)，接下來系統需要透過這份 Workout_Log.json 裡訓練者做的重量、完成次數、停用狀態等資料值，搭配當前的訓練狀態與 [Stronglift 5x5](https://stronglifts.com/stronglifts-5x5/workout-program/) 課表規則，進行邏輯與算術處理，目標是生成出下次的訓練內容。

## 函數呼叫
> **「就是因為 ChatGPT 它在做文字接龍，所以它很多時候會講一些不是事實的話。」── 李宏毅** [🔗](https://www.youtube.com/shorts/tFxs2uDwdKo)   

LLM 的原理是以機率生成文字，並不具備邏輯與算術能力，AI Agent 的其他功能必須外接函數或 API，並藉由 LLM 和這些函數溝通以啟用這些額外功能，而函數呼叫是其中一種實作方式。   
不過由於本專案使用的範例訓練課表是規則明確的 [Stronglift 5x5](https://stronglifts.com/stronglifts-5x5/workout-program/)，目前 MVP 的 LLM 還不需要呼叫工具。而之後要生成客製化課表時，LLM 就得借助工具的幫忙完成課表。

## 給訓練者下次訓練內容：所需的後端流程
Week 6 撰寫了從訓練者自然語言輸入，提取重要資訊並產生當次訓練紀錄 JSON 檔的 Prompts。產生的 Workout_Log.json 裡的值為決定下一次訓練內容的基礎。   
雖然 [Stronglift 5x5](https://stronglifts.com/stronglifts-5x5/workout-program/) 課表的規則已經成熟運行數十年，本系統需要產出的訓練內容仍必須經過後端業務流程處理邏輯與運算求得，這些函數用於回傳準確值，避免 LLM 用機率猜測出錯誤的數值胡言亂語。   
函數大多數遵守 [Stronglift 5x5](https://stronglifts.com/stronglifts-5x5/workout-program/) 的規則，不過部分的函數也能用於其他課表使用。   

### Stronglift 5x5 的基本規則
* 如果按照計畫，該運動動作完成 5 組，且每一組都完成預定重量，下次訓練目標重量 = 原重量 + 5 磅 (5 磅約為 2.5 公斤)。
* 如果沒有完成計畫的組數與次數，下次訓練維持原重量。
* 同個動作連續三次訓練都失敗，下一次降低 10% 重量。

本週我先絞盡腦汁，設計完整的後端流程(偽代碼)，下週用 Python 實作。   
### 大流程: 
    1️⃣ 根據Workout_Log.json更新Current_State.json -> 2️⃣ 根據更新後的Current_State.json決定是否下降訓練動作的重量 -> 3️⃣ 根據Workout_Log.json停用受傷部位的動作 -> 4️⃣ 輸出下次訓練安排   

### Stronglift_5x5_Rules類別裡的常數:
```Markdown
float min_weight_change_in_kilogram = 2.5   
float deload_proportion = 0.9   
String day_tag = ['A', 'B']   
```

### 每個階段的詳細函數說明
1. 根據Workout_Log.json更新Current_State.json   
Stronglift_5x5_Rules類別裡要有的函數:   
```Markdown
(1) 檢查Workout_Log.json裡，performed_exercises中is_exercise_in_program為true的物件，
若performed_sets陣列裡有>=5個物件，且每個物件的weight_in_kilograms>=(Current_State中lifts_status的對應動作的current_weight_kg)，且每個物件的performed_reps>=5，則add_weight(current_weight_kg, 2.5)且consecutive_fails設為0；除此之外consecutive_fails+=1。   
(2) add_weight(float current_weight_kg, float min_weight_change_in_kilogram): Current_State中lifts_status的對應動作的current_weight_kg+=min_weight_change_in_kilogram。   
(3) 將Current_State.json中lifts_status的物件==Workout_Log.json裡performed_exercises中is_exercise_in_program為true的物件，status設為active且last_substituted_by設為null。   
(4) 檢查Workout_Log.json裡，pre_workout_status中skipped_exercise陣列裡的物件，return它們。   
(5) 將Current_State.json中lifts_status的物件==(4) return的值，status設為frozen。   
(6) 檢查Workout_Log.json裡，performed_exercises中is_exercise_in_program為false的物件，return exercise_name和substitute_for。   
(7) update_exercise_substitution (String exercise_name, String substitute_for): 將Current_State.json中lifts_status的物件==substitute_for，其last_substituted_by設為exercise_name。   
```

2. 根據更新後的Current_State.json決定是否下降下次訓練動作的重量   
Stronglift_5x5_Rules類別裡要有的函數:   
```Markdown
(1) 判斷Current_State.json中program_state的next_workout_type，若為A，將課表中Day A的動作匯入下次訓練的動作且next_workout_type = (next_workout_type+1)%(day_tag.length)+1。   
(2) Current_State中lifts_status裡的物件，若consecutive_fails>2，則deload(current_weight_kg)且將consecutive_fails設為0。接著return current_weight_kg。   
(3) deload(float current_weight_kg, float min_weight_change_in_kilogram, float deload_proportion): current_weight_kg = current_weight_kg * deload_proportion - current_weight_kg % min_weight_change_in_kilogram 。   
(4) num_of_sets = 課表中每個動作的組數，num_of_reps =  課表中每個動作的次數。組數回傳num_of_sets；次數回傳num_of_reps。 
```  

3. 根據Workout_Log.json停用受傷部位的動作   
```Markdown
(1) 查詢資料庫中，動作的部位 =  Workout_Log.json的exhausted_or_injured_body_parts裡的物件，return這些動作。   
(2) 將函數(1) return的動作停用。   
```   
4. 輸出下次訓練安排: 動作、重量、組數、次數、是否停用。   

此外，也撰寫了 Stronglift_5x5_Program.json 和 Next_Training_Output.json 的範本。   
### Stronglift_5x5_Program
```json
{
    "_comment": "This format records the default program of Stronglift 5x5.",
    "program_id": "P000",
    
    "day": {
        "A": {
            "barbell_squat": {
                "weight_in_kilograms": 20.0,
                "number_of_sets": 5,
                "number_of_reps": 5,
                "status": "active"
            },
            "barbell_bench_press": {
                "weight_in_kilograms": 20.0,
                "number_of_sets": 5,
                "number_of_reps": 5,
                "status": "active"
            },
            "barbell_row": {
                "weight_in_kilograms": 20.0,
                "number_of_sets": 5,
                "number_of_reps": 5,
                "status": "active"
            }
        },
        "B": {
            "barbell_squat": {
                "weight_in_kilograms": 20.0,
                "number_of_sets": 5,
                "number_of_reps": 5,
                "status": "active"
            },
            "barbell_overhead_press": {
                "weight_in_kilograms": 20.0,
                "number_of_sets": 5,
                "number_of_reps": 5,
                "status": "active"
            },
            "barbell_deadlift": {
                "weight_in_kilograms": 20.0,
                "number_of_sets": 1,
                "number_of_reps": 5,
                "status": "active"
            }
        }
    }
}
```

### Next_Training_Output.json
```json
{
    "_comment": "This format records the output of next training.",
    "output_id": "O004",
    
    "date": "2026-04-07",
    "day": "B",
    "exercises": {
        "barbell_squat": {
            "weight_in_kilograms": 70.0,
            "number_of_sets": 5,
            "number_of_reps": 5,
            "status": "active"
        },
        "barbell_overhead_press": {
            "weight_in_kilograms": 20.0,
            "number_of_sets": 5,
            "number_of_reps": 5,
            "status": "frozen"
        },
        "barbell_deadlift": {
            "weight_in_kilograms": 60.0,
            "number_of_sets": 1,
            "number_of_reps": 5,
            "status": "active"
        }
    }
}
```
## 系統架構策略：最小權限與職責分離
本專案將系統拆分為三個獨立模組：   
意圖擷取 (Agent 1)：將自然語言轉為標準的 Workout_Log.json 格式。   
核心決策 (Python 後端)：處理確定性高的商業邏輯 (加減重、凍結)，產出 Next_Training_Output.json。    
語意包裝 (Agent 2)：僅擁有讀取最終 JSON 的權限，負責根據單純的資料，匯出含有情緒價值的自然語言回饋。   

## 給訓練者下次訓練內容：讓 LLM 提供情緒價值
下次訓練內容已經過確定的業務邏輯產出，LLM 須根據後端產出的 JSON (即 Next_Training_Output.json)，以自然語言和訓練者報告下次訓練內容。   
因為訓練動作資訊會以制式程式碼，根據 Next_Training_Output.json 繪製，因此 LLM 不需要自行生成表格，進而限制 LLM 的權力。   
以下是 Next_Training_Generator_System_Prompt.md 的提示詞，給予 Agent 的職責非常單純，就是白話告訴訓練者下次的訓練內容。

```Markdown
# Role & Goal: 
You are an expert and compassionate AI Training Assistant.   
Your goal is to tell the user next training based on the JSON file from the system. 

# Input Context:
The system will input `Next_Training_Output.json`, including exercises, weights, sets, and status of the exercise.

# Rules and Constraints: 
1. Tone: Simple, clear, and encouraging. **DO NOT** create a table because the system already creates one. All you need is to express the information in sentences.
2. Values: Strictly stay true to the values in the JSON file. **DO NOT** adjust the exercises or numbers.
3. About "frozen" status: If an exercise's status is "frozen", clearly inform the user that this exercise has been paused for safety or recovery. **DO NOT** encourage them to perform that specific exercise or push through the limit.
4. No medical advice: If the context implies the user is injured, recommend seeking professional medical help or a physical therapist. **DO NOT** diagnose or prescribe medical treatments. 
5. Language limitation: Always respond in Traditional Chinese (zh-TW) in the final output.
```
   
對我而言，寫好一個 Prompt 的難度不亞於寫程式碼，自認不擅長的原因，可以歸咎於**不具備良好的文字表達**，以及**沒有清晰定位 LLM 的權限**。正巧，這些也是人與人溝通、任務指派的必要條件。   
* **良好的文字表達**在於措辭是否精準，避免模糊造成誤解。如同如何讓對方馬上聽懂自己的話，且理解的內容與自己沒有出入。目前卡在對於詞彙的精確使用方式不夠清楚，導致撰寫 Prompts 時時常陷入詞窮。   
* **清晰定位 LLM 的權限**代表自己須清楚並明確界定 LLM 能做到哪些事、存取哪些資訊。如同和人溝通必須站在對方的視角，清楚對方的職責，才能避免代溝。   

## 關於這份專案與實習商討事宜
### 專案緣起
我認為自己與其他新手訓練者的差異就是我投入三年以上的時間訓練和看 Youtube 影片，這些對於已經忙碌地新進自主訓練者更難自行累積，許多剛踏入健身房的人難以持之以恆，就無法藉由運動獲得身心健康、自我追求等目標。但我依然堅持請教練不是訓練唯一的選擇，抱持著不認輸的心態持續自主訓練，不只是受限於學生預算，也更想挑戰自我。漸漸地，自己感到能去重訓的時間越來越少，加上無論多麼完善的訓練計畫，實際遇到各種突發狀況也束手無策，例如當健身房器材被占用，對一般人而言，負擔大量時間成本等候是不切實際的。此外現有的 App 所提供的客製化訓練計畫功能，其操作既枯燥又繁瑣，反倒使用筆記本更有效率，使得 App變得雞肋。   
我也曾利用 Gemini Pro 協助課表安排與進步追蹤。但即便是通用型的 AI 助理，也有一定的侷限性: 最大的麻煩就是每次我要先手動整理當天訓練的紀錄與附註，接著思考並提出一套看法，傳給 Gemini 後還需要消化和檢查是否有誤解的情況，時間成本從看影片或上教練課轉移到了「寫文章」上。不過，我也注意到處理變動狀況與提供情緒價值是生成式 AI 拿手的部分，也許從 Google Gemini 這類通用的聊天或任務處理的機器人，脫離並建立一套專門給予健身客群的訓練輔助 App，是一件值得嘗試的事，也因此開始這門專案。   
### 截至目前為止的心得
AI Training Assistant 是為了個人訓練服務的系統，我期許它能在教練的專業知識與靈活應變的彈性中取得平衡，讓自主訓練免於繁瑣的研究或是記錄。開發這門專案的過程，我對於 AI 應用的認知從單純使用機器學習模型分析資料，或只是想像著 AI 黑盒子能自行照規則做任務，直到現在對於近期實際的落地應用有了更清晰且具體的視野。   
這份專案我打算迭代開發，直到系統達到能夠客製化訓練計畫，以及能降低我和其他自主訓練者在訓練上的阻力，向永續訓練的目標邁進。 
### 產業實習嘗試
目前市面上也有許多企業，深耕數位健康與運動科技產業，追求的目標與我目前的專案一致，不過規模更龐大且經驗更豐富。在業界遇到的實際狀況是個人開發專案無法模擬的，為了讓 AI Training Assistant 能解決更多實務問題，我打算爭取相關企業實習的機會，並於大四學期期間參與。  
我是資訊管理背景的大三生，具備軟體流程、程式設計和資料流程等基礎知識，且目前正著手進行數個系統設計與開發，希望能為有相關需求且追求同樣願景的企業提供即戰力。上個月我已針對 Garmin 和工業技術研究院的學期實習缺投遞履歷，不同於大廠的成熟標準流程，中小企業的職位更彈性以面對變動的環境，在我著手調查後，發現他們的官網沒有特別徵人或是介紹公司的業務，也許對他們而言，投資徵人的成本不低。   
我找了以下幾間新創，並進行深入研究:    
* [虹映科技股份有限公司 (Joiiup)](https://www.joiiup.com/about-us): 位於新竹縣竹北市。提供數位個人全方位健康管理工具，以及企業使用的職場健康管理平台等服務。旨在結合AI數據分析等資訊科技，促進人們維持運動習慣以獲得健康。   
* [創鐸股份有限公司 (Trendonut Co., LTD.)](https://trainge.com/tw/home): 位於台中市中區。旗下的 Trainge 運動平台提供從媒合、預約課程到訓練紀錄一站式的訓練流程，客群包含一般運動者、教練和運動場館。      
以下企業並非歸類於法定中小企業的定義，但同樣有以數位科技提升健康的願景:    
* [真茂科技股份有限公司 (NETOWN CORPORATION)](https://netown.tw/about.html): 位於新北市板橋區。整合醫療、電子、資通訊等領域，提供智慧健康解決方案，和許多醫療院所、長照機構密切合作。近期推動主品牌「寶貝機」的 AI 智慧照護床系統。    

這幾家企業的共同點，在於都需要處理大量且複雜的個人健康或運動數據。我認為我在本次專案中所鍛鍊的**資料管線處理**、**邊界條件與狀態機設計**，以及**限制 LLM 權限的系統架構思維**，正是這類高度依賴資料流的企業所需要的即戰力。   
我預期能在大四學期期間，每週安排 2 ~ 3 天投入實習。希望近期有機會能與老師約個時間，請益關於上述 Fit-Tech 產業的實習方向，或是探詢是否有促成相關面試的可能。   
若老師評估我目前的技術廣度或深度尚不足以勝任這些企業的實習，也懇請老師不吝點出我缺乏的拼圖，我會盡速規畫學習路徑來補足缺陷，讓自己做好萬全準備。謝謝老師撥冗閱讀。   
   
> 🤖 與 Gemini Pro 的對話記錄連結: [🔗](https://docs.google.com/document/d/1cULJvt3Ko2Q5VuhqNMzKcZNMeEUkcqtQt-kXjUtpWfk/edit?tab=t.0)   
> 🏋️ Project Repository: [🔗](https://github.com/Darren-Dev-Repo/AI_Training_Assistant)   
> <- [Week 6](week_6.md)   
> ______[Week 8](week_8.md) ->   