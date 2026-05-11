# 2026/05/05 Week 11 進度回報：動態提示詞實作與 Gemini API 核心串接
> 🤖 與 Gemini Pro 的對話記錄連結: [🔗](https://docs.google.com/document/d/1weCMuFQa8n2daM_Y-cvmk_X9g0oa0g_aLh4PfEeZhYk/edit?tab=t.wyrl2bljur)   
> 🏋️ Project Repository: [🔗](https://github.com/Darren-Dev-Repo/AI_Training_Assistant)   
> <- [Week 10](week_10.md)  
> ______[Week 12](week_12.md) ->   

## 執行摘要 (Executive Summary)
本週專案迎來重大突破，成功從架構構想，推進至可運作的 MVP。   
本專案不採用龐大且耗費 Token 的 MCP 架構，選擇 **動態上下文注入結合輕量級工具呼叫** 的精確架構，並成功串接 Google Gemini API，實現了能在 CLI 終端機中進行多工運算的 Stronglift 5x5 智能教練核心迴圈。

## 架構決策與代幣經濟學 (Architecture & Tokenomics)
在實作 API 串接前，閱讀其他同學關於 Agent 操作外部工具方式的探討，搭配文章和影片綜合進行評估:   

* 捨棄泛用型 MCP / CLI 授權: 雖然業界常以 MCP 解決多工具場景，但經評估，本系統為垂直領域的健身助理，核心需求為「確定性」與「高安全性」。給予過大權限或過多 Context 易引發 LLM 幻覺，且 Token 消耗極大。   

* 採用 API Function Calling: 決定將商業邏輯（加重、降重、切換課表）嚴格封裝於 Node.js / TypeScript 後端，LLM 僅負責「意圖判斷與參數傳遞」。關於工具實作，已經在[上週完成細節實作](https://darren-dev-repo.github.io/ai_training_assistant/reports/week_10.html#%E5%B7%A5%E5%85%B7%E5%BA%AB%E5%AF%A6%E4%BD%9C)，本週將其打包為 myTools 匯出。   

* 效益與 Tokenomics 實測: 
採用 API Function Calling 並選用最新的 gemini-2.5-flash 模型後，經由 Google AI Studio 後台實測，   
> Input Tokens： 10.32k (主因為 System Prompt、狀態注入與多工迴圈的 Context 傳遞)   
> Output Tokens： 3.85k (LLM 僅輸出精簡的工具指令與最終回覆)   

總計成本不到 $0.002 美金（約新台幣 0.06 元）。此數據強力佐證了本專案採用的「前端決策 + 後端運算」架構，不僅徹底解決了 LLM 運算幻覺的問題，更在經濟效益上達到了完美的最佳化。  

## 動態提示詞工程實作 (Dynamic Prompting Implementation)
為了解決 LLM 容易混淆數據與規則的問題，本週實作了資料與邏輯分離的 Prompt 注入系統:     
* 領域知識 (Domain Knowledge)： 將 5x5 的課表規則撰寫成 Stronglift_5x5.md，使 LLM 易於閱讀。   
* 狀態記憶 (State Memory)： 將使用者的當前訓練目標與連續失敗次數獨立為 Current_State.json。   
* 動態組裝與行動綱領 (Actionable Rules)： 透過 Node.js 檔案系統 (fs) 動態讀取上述兩者，並在 System Prompt 中加入嚴格的「三階段狀態應對指南（成功、維持、降重）」。明確限縮 LLM 權限，並強制規範輸出格式。   

```Typescript
import * as fs from 'fs';
import * as path from 'path';
import { fileURLToPath } from 'url';

// ESM 環境下的 __dirname
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

export function buildSystemPrompt(): string {
    try {
        // 1. 動態讀取外部檔案
        const statePath = path.join(__dirname, 'Current_State.json');
        const rulesPath = path.join(__dirname, 'Stronglift_5x5_Program.md');

        const currentStateString = fs.readFileSync(statePath, 'utf-8');
        const exerciseRulesString = fs.readFileSync(rulesPath, 'utf-8');

        // 2. 組裝最終的 Prompt 字串
const systemPrompt = `
# Role & Task (角色與任務)
你是一位嚴格、專業且充滿熱忱的「Stronglift 5x5 專屬健力教練 Agent」。
你的任務是根據使用者的訓練回報，嚴格比對 5x5 規則與使用者當前狀態，判斷其挑戰成功或失敗，並呼叫對應的工具來計算下次重量。

# Domain Knowledge: 訓練課表與規則 (Stronglift 5x5)
${exerciseRulesString}

# Context: 訓練者當前狀態 (Current State)
以下是訓練者目前的各項動作狀態與目標重量、連續失敗次數（consecutive_fails）：
${currentStateString}

# Actionable Rules (執行邏輯與工具呼叫規範)
1. 【比對數據】：逐一對比使用者回報的狀況與【當前狀態】中的 \`current_weight_kg\` 與 \`consecutive_fails\`。注意：跳過沒做也視同失敗。
2. 【判斷與工具呼叫】：
   - 🟢 成功：完成所有規定組數與次數。請呼叫 \`addWeight\` 工具。
   - 🟡 失敗但維持重量：未達標，且該動作當前的 \`consecutive_fails\` **小於 2**。請**不要**呼叫加重或降重工具，下次維持原重量。（但需在回覆中安撫使用者）。
   - 🔴 連續卡關降重：未達標，且該動作當前的 \`consecutive_fails\` **等於 2**（代表加上今天是第 3 次）。請呼叫 \`deload\` 工具。
3. 【切換訓練日】：每次訓練結束，【必須】呼叫 \`getNextWorkoutType\` 工具確認下次是 A 日還是 B 日。
4. 【絕對限制】：嚴禁你自己進行加減乘除計算重量！

# Output Format (回覆語氣與格式規範)
工具回傳結果後，請以口語、鼓勵的語氣回覆使用者，並且【必須】遵守以下格式：
- 根據 \`getNextWorkoutType\` 的結果，明確指出下一次是哪一個訓練日。
- 對照課表（Domain Knowledge），列出下一次訓練日【所有必須執行的動作名稱】與【目標重量】。
`;
        return systemPrompt;
    } catch (error) {
        console.error("讀取狀態或規則檔案時發生錯誤：", error);
        throw error;
    }
}
```

## Gemini API 串接與多工迴圈突破 (API Connection & Tool Calling Deadlock Resolution)
在串接 Gemini 2.5 Flash 模型時，由於連續工具呼叫，造成一般單程對話無法處理的阻塞問題。   
具體情況為，當使用者一次回報多項動作的狀況，LLM 會試圖連續呼叫多次 deload 工具，導致原有程式碼在等待純文字回應時發生中斷，不會成功產生回應。   
因此於 TypeScript 中實作了多工處理迴圈與防禦性編程。系統能攔截並批量處理 LLM 丟出的多個工具請求，並具備自動捕捉不存在的工具呼叫的能力。   
```Typescript
async function startAgent() {
    const rl = readline.createInterface({ input, output });
    
    // 建立一個帶有歷史記憶的對話 Session
    const chatSession = model.startChat();

    console.log("==================================================");
    console.log("🏋️ Stronglift 5x5 智能教練已上線！(輸入 exit 離開)");
    console.log("==================================================\n");

    while (true) {
        const userInput = await rl.question('👤 你：');
        if (userInput.toLowerCase() === 'exit') break;

        process.stdout.write('💪🤖 教練思考中...');

        try {
            // 第一步：把使用者的話傳給 LLM
            let result = await chatSession.sendMessage(userInput);
            let response = result.response;
            let functionCalls = response.functionCalls();

            // 多工迴圈 + 防禦性檢查
            while (functionCalls && functionCalls.length > 0) {
                // 準備一個陣列，用來裝這回合所有工具算出來的答案
                const functionResponses = [];

                for (const call of functionCalls) {
                    // 確保 call 不是 undefined
                    if (!call) continue;

                    // 確保這個工具存在於 myTools 裡
                    const toolFunction = myTools[call.name as keyof typeof myTools];
                    
                    if (!toolFunction) {
                        console.error(`\n[系統錯誤] LLM 試圖呼叫不存在的工具：${call.name}`);
                        // 如果工具不存在，依然要回傳一個錯誤給 LLM，不然 API 會報錯說「你少回傳了結果」
                        functionResponses.push({
                            functionResponse: {
                                name: call.name,
                                response: { error: `找不到名為 ${call.name} 的工具，請確認。` }
                            }
                        });
                        continue; 
                    }

                    process.stdout.write(`\n[系統後台] 執行工具 ${call.name}...`);
                    
                    try {
                        // 第三步：呼叫對應的 TypeScript 函數
                        const apiResponse = toolFunction(call.args);
                        
                        functionResponses.push({
                            functionResponse: {
                                name: call.name,
                                response: apiResponse
                            }
                        });
                    } catch (err) {
                        console.error(`\n[系統錯誤] 工具 ${call.name} 執行時崩潰！`);
                        functionResponses.push({
                            functionResponse: {
                                name: call.name,
                                response: { error: "工具內部執行失敗" }
                            }
                        });
                    }
                }

                process.stdout.write('\n💪🤖 教練正在計算數據與彙整課表...');

                // 第四步：把所有算完的答案「一次打包」還給 LLM
                result = await chatSession.sendMessage(functionResponses);
                response = result.response;
                
                // 檢查教練是不是還有話要說，或者又要呼叫新工具（更新迴圈條件）
                functionCalls = response.functionCalls();
            }

            // 第五步：當 while 迴圈結束，代表教練真正準備好開口了！
            console.log(`\n\n💪🤖 教練：${response.text()}\n`);

        } catch (error) {
            console.error("\n發生錯誤：", error);
        }
    }

    rl.close();
    console.log("系統關閉，下次訓練見！");
}

startAgent();
```   
## 下週規劃
至此，System Prompt、LLM、Plugin 和 Workflow 的基礎已經完成，第 12 週將專注於:   
* Memory: 實作 TypeScript 工具，將 LLM 運算完的最新重量寫回 Current_State.json，完成實體記憶迴圈。   
* Guardrails: 於 Prompt 中加入對話邊界限制，拒絕處理非健身領域之提問。   
* Knowledge： 建立基礎動作資料庫，為 Week 13 的 RAG 雛形做準備。   


> 🤖 與 Gemini Pro 的對話記錄連結: [🔗](https://docs.google.com/document/d/1weCMuFQa8n2daM_Y-cvmk_X9g0oa0g_aLh4PfEeZhYk/edit?tab=t.wyrl2bljur)   
> 🏋️ Project Repository: [🔗](https://github.com/Darren-Dev-Repo/AI_Training_Assistant)   
> <- [Week 10](week_10.md) 
> ______[Week 12](week_12.md) ->