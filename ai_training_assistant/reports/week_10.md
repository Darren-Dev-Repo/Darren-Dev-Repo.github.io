# 2026/04/28 Week 10 進度回報：Typescript 函式庫與 LLM 工具規格
> 🤖 與 Gemini Pro 的對話記錄連結: [🔗](https://docs.google.com/document/d/1cULJvt3Ko2Q5VuhqNMzKcZNMeEUkcqtQt-kXjUtpWfk/edit?tab=t.0)   
> 🏋️ Project Repository: [🔗](https://github.com/Darren-Dev-Repo/AI_Training_Assistant)   
> <- [Week 7](week_7.md)  

## 學習重心與專案架構調整(含工作流概覽)
原定的

而 [Week 7 當時的流程設計](https://darren-dev-repo.github.io/ai_training_assistant/reports/week_7.html#%E7%B3%BB%E7%B5%B1%E6%9E%B6%E6%A7%8B%E7%AD%96%E7%95%A5%E6%9C%80%E5%B0%8F%E6%AC%8A%E9%99%90%E8%88%87%E8%81%B7%E8%B2%AC%E5%88%86%E9%9B%A2)
```Markdown
## 系統架構策略：最小權限與職責分離
本專案將系統拆分為三個獨立模組：   
意圖擷取 (Agent 1)：將自然語言轉為標準的 Workout_Log.json 格式。   
核心決策 (Python 後端)：處理確定性高的商業邏輯 (加減重、凍結)，產出 Next_Training_Output.json。    
語意包裝 (Agent 2)：僅擁有讀取最終 JSON 的權限，負責根據單純的資料，匯出含有情緒價值的自然語言回饋。   

## 給訓練者下次訓練內容：讓 LLM 提供情緒價值
下次訓練內容已經過確定的業務邏輯產出，LLM 須根據後端產出的 JSON (即 Next_Training_Output.json)，以自然語言和訓練者報告下次訓練內容。   
因為訓練動作資訊會以制式程式碼，根據 Next_Training_Output.json 繪製，因此 LLM 不需要自行生成表格，進而限制 LLM 的權力。   
```


## LLM Function Calling 細節
在 AI Agent 的系統裡，函數可以被歸類為工具的一種，在[Week 7 進度報告文件](https://darren-dev-repo.github.io/ai_training_assistant/reports/week_7.html#%E7%B5%A6%E8%A8%93%E7%B7%B4%E8%80%85%E4%B8%8B%E6%AC%A1%E8%A8%93%E7%B7%B4%E5%85%A7%E5%AE%B9%E6%89%80%E9%9C%80%E7%9A%84%E5%BE%8C%E7%AB%AF%E6%B5%81%E7%A8%8B) 也提過工具/函數的用途是提供 LLM 正確答案，讓 Agent 勝任除了單純聊天之外的更多任務。而這次則要說明工具呼叫具體是如何做到的。   
### 工具呼叫流程
平時使用者使用 Gemini 或是其他 AI 助理時，看似是使用者提出需求，AI 給出回應，是一問一答的模式。然而實際上，這中間會經歷[數次系統/開發者和模型間的互動](https://developers.openai.com/api/docs/guides/function-calling#the-tool-calling-flow)。其中最簡單的架構也至少需要兩次來回的溝通 (有點像網路的握手但又不太一樣)：   
* 第一次溝通: 由系統/開發者端傳遞**工具** (含定義與規格等) 和**文本**給模型，模型根據文本內容選擇需要的工具，並回傳使用該工具的指令給系統。   
* 第二次溝通: 系統收到指令後即使用工具以獲得結果，接著便會傳遞**結果值**與**先前的文本**給模型，讓模型根據這些值生成最終結果再傳回系統。   

其實，在工具呼叫出現前，就是得先問一次模型某個任務可以用哪一個手邊工具解決，接著根據模型的回答自行解決後，再拿著答案和前情提要問一次模型。對使用者來說，工具呼叫流程不過是自動化了中間選擇和操作工具的流程。      
### 函數規格
如同說明書一般，函數規格清楚地向模型說明函數的**使用時機**、**參數和其意義**等，通常以我們再熟悉不過的 json 格式呈現。內部的架構可以參考 [OpenAI Developers 文章](https://developers.openai.com/api/docs/guides/function-calling#defining-functions)的說明，其依循著 JSON schema 的定義，通用於各種 Agent 的開發。   

## 工具庫實作
以 TypeScript 實作**增重**、**切換訓練日日期**和**降重**的函式。   
```TypeScript
interface ITrainingTools {
    addWeight(currentWeightKg: number, minWeightChangeInKg: number): number;
    getNextWorkoutType(currentWorkoutType: string, dayTag: string[]): string;
    deload(currentWeightKg: number, minWeightChangeInKg: number, deloadPercentage: number): number;
}

class AgentTrainingTools implements ITrainingTools {
    addWeight(currentWeightKg: number, minWeightChangeInKg: number): number {
        return currentWeightKg + minWeightChangeInKg;
    }

    getNextWorkoutType(currentWorkoutType: string, dayTag: string[]): string {
        const currentIndex: number = dayTag.indexOf(currentWorkoutType);

        /* 防呆機制：如果傳入一個不在陣列裡的字串
           我們就強迫訓練者從第一天開始練 */
        if (currentIndex === -1) {
            return dayTag[0] || "A";
        }

        const nextIndex = (currentIndex + 1) % dayTag.length;
        return dayTag[nextIndex] || "A";
    }

    deload(currentWeightKg: number, minWeightChangeInKg: number, deloadPercentage: number): number {
        let rawWeight: number = currentWeightKg * deloadPercentage;
        // Make the final weight be a multiple of the weight of the lightest plate in the gym 
        return Math.floor(rawWeight / minWeightChangeInKg) * minWeightChangeInKg;
    }

}
```
   
使用類別繼承介面有利於未來擴充其他額外功能，或是其他課表專屬的功能。   

## 工具說明規格
接著根據先前實作的函數，分別撰寫其對應的 json 函數規格。   
### /tool_definition/Stronglift_5x5/addWeight.json
```json
{
  "type": "function",
  "function": {
    "name": "addWeight",
    "description": "當使用者完成 5x5 訓練，表示他挑戰成功時，使用此工具計算下一次的訓練重量。",
    "parameters": {
      "type": "object",
      "properties": {
        "currentWeightKg": {
          "type": "number",
          "description": "使用者今天完成的重量，單位為公斤 (kg)。"
        },
        "minWeightChangeInKg": {
          "type": "number",
          "description": "要增加的重量，通常是 2.5 kg。"
        }
      },
      "required": [
        "currentWeightKg",
        "minWeightChangeInKg"
      ]
    }
  }
}
```
### /tool_definition/Stronglift_5x5/getNextWorkoutType.json
```json
{
  "type": "function",
  "function": {
    "name": "getNextWorkoutType",
    "description": "需要知道使用者下一次訓練是課表的哪一天時，使用此工具取得訓練日的名稱。",
    "parameters": {
      "type": "object",
      "properties": {
        "currentWorkoutType": {
          "type": "string",
          "description": "使用者最近一次完成的訓練日的名稱。"
        },
        "dayTag": {
          "type": "array",
          "description": "存放使用者訓練課表中，所有訓練日的名稱 (例如 ['A', 'B'])",
          "items": {
            "type": "string"
          }
        }
      },
      "required": [
        "currentWorkoutType",
        "dayTag"
      ]
    }
  }
}
```
### /tool_definition/Stronglift_5x5/deload.json
```json
{
  "type": "function",
  "function": {
    "name": "deload",
    "description": "使用者未完成預計重量，挑戰失敗時，使用此工具計算下一次的訓練重量。",
    "parameters": {
      "type": "object",
      "properties": {
        "currentWeightKg": {
          "type": "number",
          "description": "使用者今天完成的重量，單位為公斤 (kg)。"
        },
        "minWeightChangeInKg": {
          "type": "number",
          "description": "要增加的重量，通常是 2.5 kg。"
        },
        "deloadPercentage": {
          "type": "number",
          "description": "降低重量的比率。"
        }
      },
      "required": [
        "currentWeightKg",
        "minWeightChangeInKg",
        "deloadPercentage"
      ]
    }
  }
}
```

## 下一次進度
* 建立工具呼叫 api (Primary)   
* 撰寫一部分的工具呼叫 Prompt (Secondary)   

   
> 🤖 與 Gemini Pro 的對話記錄連結: [🔗](https://docs.google.com/document/d/1cULJvt3Ko2Q5VuhqNMzKcZNMeEUkcqtQt-kXjUtpWfk/edit?tab=t.0)   
> 🏋️ Project Repository: [🔗](https://github.com/Darren-Dev-Repo/AI_Training_Assistant)   
> <- [Week 7](week_7.md)     