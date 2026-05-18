# 2026/05/12 Week 12 進度回報：Memory、Guardrails 與 Knowledge Base 的實作
> 🤖 與 Gemini Pro 的對話記錄連結: [🔗](https://docs.google.com/document/d/1RDTNOVAKQ0UHvHKF0GPSnPwdCxWua6g45BsxafRtHXM/edit?tab=t.0)   
> 🏋️ Project Repository: [🔗](https://github.com/Darren-Dev-Repo/AI_Training_Assistant)   
> <- [Week 11](week_11.md)  

## 測試導向開發
　　開發的過程中，為了驗證系統能在每種情況都會正確執行結果，以不斷的測試作為建設的依據，並藉此發現設計期間未考量到的疏漏。   
　　拿以下的情境作舉例，我這次在工具庫實作了增加失敗次數的工具，並模擬訓練者輸入以下文本進行測試:    

```Plaintext
健身房充斥著下班人潮，好險深蹲架還有空位，不過晚點還要參加其他課程，今天只有 40 分鐘可以訓練。後來，我深蹲做完了 65 公斤 5 組，每組 5 下；臥推練到第五組時沒什麼力了，就改用 25 公斤做完 8 下；划船只做了 40 公斤 3 組，每組 5 下。
```
   
　　根據先前紀錄和訓練者提供的敘述，深蹲有達成預計的訓練標準，而 LLM 也確實判定成功，呼叫工具進行加重，然而同時它又呼叫了計算失敗次數的工具，導致明明成功進步了，卻留下了失敗紀錄。因此，我在工具的[規格說明書](https://github.com/Darren-Dev-Repo/AI_Training_Assistant/blob/main/AgentToolsDescription.ts)內加入了 reasoning，讓 LLM 呼叫工具前先自己解釋如此決策的理由。   
　　另外，測試的過程中也務必要保持腦袋清楚，一次的測試中，我忘記目標重量已經因為進步而增加了，依然拿舊的模擬 prompt 給系統，當結果不符預期時還以為出錯了，結果是自己搞了個烏龍。   

## 記憶機制
　　在 Week 11 的進度中，系統已經能根據當下的對話給出訓練建議。然而，這就像是一位只靠腦力記住學員狀態的教練，他當下雖然算出了下次深蹲要加重到 72.5 公斤，但如果沒有把這個數字寫進筆記本裡，下一次上課時，他依然會從舊的進度開始算起。在我們的系統中，筆記本就是 Current_Status.json。如果沒有將更新後的數據覆寫回檔案，系統就會永遠讀取過期的舊資料。   
　　因此，我將檔案讀寫的機制，直接封裝進 Agent 呼叫的工具內部，隨著每次訓練結束，Current_Status.json 中需要更動的地方都能同步被覆寫到最新的狀態，進而具備記憶的功能，讓系統能永續運作。   
　　在工具的[規格說明書](https://github.com/Darren-Dev-Repo/AI_Training_Assistant/blob/main/AgentToolsDescription.ts)中，強制 LLM 必須額外傳入 exerciseName (動作名稱)。這樣後端系統才能精準知道現在要更新的是深蹲、臥推還是划船。   

```Typescript
export const agentTools = [
  {
    type: "function",
    function: {
      name: "addWeight",
      description: "當使用者完成 5x5 訓練，表示他挑戰成功時，使用此工具計算下一次的訓練重量。",
      parameters: {
        type: "object",
        properties: {
          exerciseName: {
              type: "string",
              description: "使用者執行的動作名稱，必須完全符合 Current_State.json 中的 key (例如: barbell_squat, bench_press 等)。"
          },
          reasoning: { 
            type: "string", 
            description: "在決定呼叫此工具前，請詳細寫下你的比對過程。例如：'深蹲目標為70kg 5x5，使用者實際完成70kg 5組5下，完全達標，故呼叫 addWeight'。"
          },
          currentWeightKg: {
            type: "number",
            description: "使用者今天完成的重量，單位為公斤 (kg)。"
          },
          minWeightChangeInKg: {
            type: "number",
            description: "要增加的重量，通常是 2.5 kg。"
          }
        },
        required: ["exerciseName", "reasoning", "currentWeightKg", "minWeightChangeInKg"]
      }
    }
  },
  ...
```
   
　　在 [TypeScript 的工具函式](https://github.com/Darren-Dev-Repo/AI_Training_Assistant/blob/main/AgentTrainingTools.ts)中，實作了檔案覆寫邏輯。當 LLM 決定加重、降重或是更新日期時，Node.js 後台會瞬間讀取、更新對應的動作數據，並存回 JSON 檔中。   

```Typescript
export const myTools: Record<string, Function> = {
    addWeight: (args: any) => {
        const nextWeight = args.currentWeightKg + args.minWeightChangeInKg;
        console.log(`\n[系統後台] 更新 ${args.exerciseName} 重量：${args.currentWeightKg} -> ${nextWeight}`);
        
        const currentState = JSON.parse(fs.readFileSync(stateFilePath, 'utf-8'));
        if (currentState.lifts_status[args.exerciseName]) {
            currentState.lifts_status[args.exerciseName].current_weight_kg = nextWeight;
            currentState.lifts_status[args.exerciseName].consecutive_fails = 0;            
        }

        fs.writeFileSync(stateFilePath, JSON.stringify(currentState, null, 4), 'utf-8');

        return { nextWeight: nextWeight };
    },
    ...
```
   
　　這樣就能在 LLM 呼叫工具時，讓後端自動地偷偷更新資料，而 LLM 也不需要知道。就像是助理自動把數字細節都記錄好了，教練只需要看筆記本裡記錄好的結果給予建議。   

## 防護措施
　　一名專業的教練是不會也不應該回應非自己業務範圍的客戶需求，因此我在 SystemPrompt.ts 補上了一道防線。   

```Markdown
7. 【安全護欄】：你的唯一身分是健身教練。若使用者詢問與 Stronglift 5x5、健身動作、營養或恢復【無關】的問題（例如：寫程式、天氣、政治、歷史），請禮貌但堅定地拒絕回答，並將話題引導回今天的訓練計畫上。
```

## 知識庫
　　我很幸運地找到了 [yuhonas 提供的 free-exercise-db 開源動作資料 (Unlicense license: 可任意下載、使用與修改)](https://github.com/yuhonas/free-exercise-db)，考量到內部的檔案 (不包含圖片) 只有5 MB，且作者已經超過一年沒有更新，便直接下載 exercise.json 作為本專案的知識庫。由於目前專案只需要讓教練做簡單的動作引導查詢，尚不須建置資料庫，只需要透過 TS 函式取得 json 裡的資料即可。   
　　不過在實際測試時遇到了小插曲，教練查詢槓鈴肩推的知識時，回報資料庫裡並沒有記錄這個動作。原來，知識庫裡槓鈴肩推使用的名字為 Military Press，而不是 Overhead Press 或 Shoulder Press，導致 LLM 使用這些關鍵字查詢時，找不到 Standing Military Press 的結果。因此我在新增的 queryExercise 工具裡又補上了同義詞的對映，讓 LLM 給出的動作關鍵詞能轉換為可能的相同名稱。  

```Typescript
    queryExercise: (args: { keyword: string }) => {
        console.log(`\n[系統後台] 檢索動作資料庫，關鍵字：${args.keyword}`);
        
        try {
            
            let searchTarget = args.keyword.toLowerCase();
            // Alias Mapping
            const aliasMap: Record<string, string> = {
                "overhead press": "military press",
                "barbell overhead press": "military press",
                "shoulder press": "military press",
            };

            // 如果使用者的關鍵字在字典裡有對應，就強制轉換成資料庫看得懂的名稱
            for (const [alias, realName] of Object.entries(aliasMap)) {
                if (searchTarget.includes(alias)) {
                    console.log(`[系統後台] 觸發同義詞轉換：${searchTarget} -> ${realName}`);
                    searchTarget = realName;
                    break;
                }
            }

            const foundExercise = staticExerciseDB.find((ex: any) => 
                ex.name.toLowerCase().includes(searchTarget) ||
                ex.id.toLowerCase().includes(searchTarget)
            );

            if (foundExercise) {
                return {
                    name: foundExercise.name,
                    primaryMuscles: foundExercise.primaryMuscles,
                    instructions: foundExercise.instructions
                };
            } else {
                return { error: `資料庫中找不到與 "${args.keyword}" 相關的動作。` };
            }
        } catch (error) {
            console.error("[系統錯誤] 檢索動作時發生異常", error);
            return { error: "檢索系統發生異常。" };
        }
    }
```

## 成果
```Plaintext
[系統啟動] 成功載入動作資料庫，共 873 筆資料入列。
==================================================
🏋️ Stronglift 5x5 智能教練已上線！(輸入 exit 離開)
==================================================

👤 你：健身房充斥著下班人潮，好險深蹲架還有空位，不過晚點還要參加其他課程，今天只有 40 分鐘可以訓練。後來，我深蹲做完了 65 公斤 5 組，每組 5 下；臥推練到第五組時沒什麼力了，就改 用 25 公斤做完 8 下；划船只做了 40 公斤 3 組，每組 5 下。
💪🤖 教練思考中...
[系統後台] 執行工具 addWeight...
[系統後台] 更新 barbell_squat 重量：65 -> 67.5

[系統後台] 執行工具 recordFail...
[系統後台] 紀錄 bench_press 失敗！目前連續失敗次數：1

[系統後台] 執行工具 recordFail...
[系統後台] 紀錄 barbell_row 失敗！目前連續失敗次數：1

[系統後台] 執行工具 getNextWorkoutType...
[系統後台] 計算下次訓練日：切換為 B

💪🤖 教練正在計算數據與彙整課表...

💪🤖 教練：太棒了！儘管時間緊迫，你還是完成了今天的訓練，這種堅持非常值得肯定！

根據你的回報，我來幫你分析一下今天的成果並規劃下次訓練：

*   **槓鈴深蹲**：你以 65 公斤完成了 5 組 5 下，表現非常出色！這代表你挑戰成功，下次訓練時深蹲的目標重量將增加到 **67.5 公斤**！
*   **槓鈴臥推**：臥推你在第五組力竭，改用 25 公斤完成，未能達到 30 公斤 5 組 5 下的目標。這次算是一次失敗，但沒關係，這是進步的必經之路！下次臥推請繼續維持 **30 公斤**，我們再來挑戰一次！
*   **槓鈴划船**：划船你完成了 40 公斤 3 組 5 下。雖然重量比上次高，但沒有完成規定的 5 組，所以也算挑戰失敗。下次划船請繼續維持 **35 公斤**，專注於完成所有組數！

別氣餒！訓練總會有起有落，重點是每一次的堅持和從失敗中學習。

下一次你的訓練日是 **B 日**。你需要完成以下動作：
*   **槓鈴深蹲**：目標 **67.5 公斤**，5 組 x 5 下
*   **槓鈴肩推**：目標 **15 公斤**，5 組 x 5 下
*   **槓鈴硬舉**：目標 **60 公斤**，1 組 x 5 下

期待你下次的表現！保持專注，穩紮穩打，你會越來越強壯的！


👤 你：槓鈴肩推有什麼技巧？有時候會有點卡。
💪🤖 教練思考中...
[系統後台] 執行工具 queryExercise...
[系統後台] 檢索動作資料庫，關鍵字：barbell overhead press
[系統後台] 觸發同義詞轉換：barbell overhead press -> military press

💪🤖 教練正在計算數據與彙整課表...

💪🤖 教練：槓鈴肩推 (Barbell Overhead Press) 是一個非常棒的動作，可以有效鍛鍊你的肩膀肌群！你提到有時候會有點卡，這很常見，調整一下技巧通常會有很大的幫助。

根據我的經驗，以下是一些槓鈴肩推的重點和技巧，可以幫助你更穩定、更有力地完成動作：

1.  **起始姿勢與握距**：
    *   將槓鈴從架上取出，放置在鎖骨上方，胸部上緣的位置。
    *   雙手握距略寬於肩，手腕保持中立，不要向後翻腕。手肘應在槓鈴下方，形成一個穩固的支撐架。
    *   雙腳約與肩同寬，腳尖可以稍微外八，保持穩定。

2.  **核心收緊**：
    *   在準備推起槓鈴前，深深吸一口氣，收緊你的核心（腹部和下背）。想像你要被揍一拳，肚子瞬間變硬的感覺。這能提供脊柱穩定性，避免下背過度拱起。
    *   臀部也要夾緊，這會幫助你保持全身的張力，像一個堅固的柱子。

3.  **推舉路徑**：
    *   推起槓鈴時，讓槓鈴盡可能垂直向上移動。想像你的頭在槓鈴「後面」經過，而不是讓槓鈴繞過你的頭。
    *   當槓鈴推過頭頂時，將頭稍微向前探，讓槓鈴最終停在你的頭部正上方，與耳朵呈一直線。這能確保槓鈴的重心在支撐點上，更省力。
    *   在頂部時，完全鎖定肘關節，將槓鈴推向天空，感受肩部肌肉的收縮。

4.  **下放控制**：
    *   下放時要保持控制，不要讓槓鈴自由落下。慢慢將槓鈴放回起始位置，再次吸氣準備下一次推舉。

5.  **呼吸節奏**：
    *   推起時吐氣，下放時吸氣。

**幾個常見「卡住」的原因與解決方案：**

*   **核心不穩**：如果推舉時下背容易拱起，或身體晃動，就是核心不夠穩定。務必在推舉前深吸一口氣並繃緊核心。
*   **槓鈴路徑不對**：如果槓鈴推舉時向前或向後晃動，表示槓鈴沒有走直線。試著推起時稍微把頭往後仰一點，讓槓鈴通過，然後再把頭向前探。
*   **肩關節活動度不足**：如果你的肩關節活動度不佳，可能會難以將槓鈴推到完全鎖定的位置。可以多做一些肩部伸展和活動度訓練。
*   **重量過重**：有時候「卡住」最直接的原因就是重量超過你當前的能力。寧可使用較輕的重量，以完美的姿勢完成動作，也不要用過重的重量導致受傷或姿勢跑掉。

希望這些建議能幫助你改善槓鈴肩推的表現！記住，多加練習，並在訓練中專注於感受肌肉發力，你會越來越熟練的。下次訓練時可以試著把這些技巧應用進去！

👤 你：那我等等的心靈課程要帶什麼顏色的背包好呢？
💪🤖 教練思考中...

💪🤖 教練：抱歉，我的職責是專注於你的 Stronglift 5x5 訓練計畫，提供健力相關的指導和建議。關於心靈課程要帶什麼顏色的背包，這超出了我能提供的範圍。

讓我們把注意力放回你的訓練上吧！你對於下次 B 日的訓練動作，例如槓鈴深蹲、槓鈴肩推或槓鈴硬舉，還有沒有其他疑問，或者需要我提供什麼幫助呢？
```
   
　　可以看到教練已經能勝任自己的職責，並準備好當一輩子的教練了 (咦？)   

## 接下來的規畫
　　有鑑於本週的系統已經具備了所有一個 AI App 會擁有的條件 (System Prompt, LLM, Plugins, Workflow, Knowledge, Memory)，下一週 (Week 13) 將**研究 Tokenoconomics**，並**模擬、統整訓練者使用功能時的花費**，最後預期**完成期末報告**。當週不會撰寫 GitHub Page，取而代之，我會直接繳交期末報告的 pdf 檔案供助教和老師查閱。   
　　Week 14 將會將這份專案做一個收尾，具體會**實作長期的歷史紀錄**、並抽象打包成**可以給訓練者使用的 App**，最後完成部署。   
　　Week 14 過後到期末為止的時間，我會實際使用這個 App 輔助我的訓練，並記錄我的使用情況。   

> 🤖 與 Gemini Pro 的對話記錄連結: [🔗](https://docs.google.com/document/d/1RDTNOVAKQ0UHvHKF0GPSnPwdCxWua6g45BsxafRtHXM/edit?tab=t.0)   
> 🏋️ Project Repository: [🔗](https://github.com/Darren-Dev-Repo/AI_Training_Assistant)   
> <- [Week 11](week_11.md) 