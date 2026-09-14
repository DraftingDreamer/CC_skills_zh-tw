---
source_file: schemas.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 8e8876180a8989b406a4d3edddf875b04cdfd5805cc8616686d552b11ce4455f
translated_at: 2026-08-22
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`schemas.md`](schemas.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
<!-- /translation-header -->

# JSON 綱要

本文件定義 skill-creator 使用的 JSON 綱要。

---

## evals.json

定義某 skill 的 eval 集合。位於 skill 目錄內的 `evals/evals.json`。

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "User's example prompt",
      "expected_output": "Description of expected result",
      "files": ["evals/files/sample1.pdf"],
      "expectations": [
        "The output includes X",
        "The skill used script Y"
      ]
    }
  ]
}
```

**欄位說明：**
- `skill_name`：與 skill frontmatter 相符的名稱
- `evals[].id`：唯一整數識別碼
- `evals[].prompt`：要執行的任務
- `evals[].expected_output`：人類可讀的成功描述
- `evals[].files`：可選的輸入檔案路徑清單（相對於 skill 根目錄）
- `evals[].expectations`：可驗證陳述的清單

---

## history.json

追蹤改進模式下的版本演進。位於工作區根目錄。

```json
{
  "started_at": "2026-01-15T10:30:00Z",
  "skill_name": "pdf",
  "current_best": "v2",
  "iterations": [
    {
      "version": "v0",
      "parent": null,
      "expectation_pass_rate": 0.65,
      "grading_result": "baseline",
      "is_current_best": false
    },
    {
      "version": "v1",
      "parent": "v0",
      "expectation_pass_rate": 0.75,
      "grading_result": "won",
      "is_current_best": false
    },
    {
      "version": "v2",
      "parent": "v1",
      "expectation_pass_rate": 0.85,
      "grading_result": "won",
      "is_current_best": true
    }
  ]
}
```

**欄位說明：**
- `started_at`：改進開始時的 ISO 時間戳記
- `skill_name`：正在改進的 skill 名稱
- `current_best`：最佳效能版本的識別碼
- `iterations[].version`：版本識別碼（v0、v1……）
- `iterations[].parent`：衍生自的父版本
- `iterations[].expectation_pass_rate`：評分後的通過率
- `iterations[].grading_result`："baseline"、"won"、"lost" 或 "tie"
- `iterations[].is_current_best`：此版本是否為目前最佳版本

---

## grading.json

評分器 agent 的輸出。位於 `<run-dir>/grading.json`。

```json
{
  "expectations": [
    {
      "text": "The output includes the name 'John Smith'",
      "passed": true,
      "evidence": "Found in transcript Step 3: 'Extracted names: John Smith, Sarah Johnson'"
    },
    {
      "text": "The spreadsheet has a SUM formula in cell B10",
      "passed": false,
      "evidence": "No spreadsheet was created. The output was a text file."
    }
  ],
  "summary": {
    "passed": 2,
    "failed": 1,
    "total": 3,
    "pass_rate": 0.67
  },
  "execution_metrics": {
    "tool_calls": {
      "Read": 5,
      "Write": 2,
      "Bash": 8
    },
    "total_tool_calls": 15,
    "total_steps": 6,
    "errors_encountered": 0,
    "output_chars": 12450,
    "transcript_chars": 3200
  },
  "timing": {
    "executor_duration_seconds": 165.0,
    "grader_duration_seconds": 26.0,
    "total_duration_seconds": 191.0
  },
  "claims": [
    {
      "claim": "The form has 12 fillable fields",
      "type": "factual",
      "verified": true,
      "evidence": "Counted 12 fields in field_info.json"
    }
  ],
  "user_notes_summary": {
    "uncertainties": ["Used 2023 data, may be stale"],
    "needs_review": [],
    "workarounds": ["Fell back to text overlay for non-fillable fields"]
  },
  "eval_feedback": {
    "suggestions": [
      {
        "assertion": "The output includes the name 'John Smith'",
        "reason": "A hallucinated document that mentions the name would also pass"
      }
    ],
    "overall": "Assertions check presence but not correctness."
  }
}
```

**欄位說明：**
- `expectations[]`：附有證據的已評分期望條件
- `summary`：彙總的通過／未通過計數
- `execution_metrics`：工具使用量與輸出大小（來自執行器的 metrics.json）
- `timing`：掛鐘時間（來自 timing.json）
- `claims`：從輸出提取並驗證的宣稱
- `user_notes_summary`：執行器標注的問題
- `eval_feedback`：（可選）eval 的改進建議，僅在評分器識別出值得提出的問題時出現

---

## metrics.json

執行器 agent 的輸出。位於 `<run-dir>/outputs/metrics.json`。

```json
{
  "tool_calls": {
    "Read": 5,
    "Write": 2,
    "Bash": 8,
    "Edit": 1,
    "Glob": 2,
    "Grep": 0
  },
  "total_tool_calls": 18,
  "total_steps": 6,
  "files_created": ["filled_form.pdf", "field_values.json"],
  "errors_encountered": 0,
  "output_chars": 12450,
  "transcript_chars": 3200
}
```

**欄位說明：**
- `tool_calls`：各工具類型的呼叫計數
- `total_tool_calls`：所有工具呼叫的總和
- `total_steps`：主要執行步驟數量
- `files_created`：已建立輸出檔案的清單
- `errors_encountered`：執行過程中的錯誤數量
- `output_chars`：輸出檔案的總字元數
- `transcript_chars`：執行記錄的字元數

---

## timing.json

某次執行的掛鐘時間。位於 `<run-dir>/timing.json`。

**擷取方式：** 當 sub-agent 任務完成時，任務通知包含 `total_tokens` 與 `duration_ms`。請立即儲存這些資料——它們不會被持久化到任何其他地方，事後也無法復原。

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3,
  "executor_start": "2026-01-15T10:30:00Z",
  "executor_end": "2026-01-15T10:32:45Z",
  "executor_duration_seconds": 165.0,
  "grader_start": "2026-01-15T10:32:46Z",
  "grader_end": "2026-01-15T10:33:12Z",
  "grader_duration_seconds": 26.0
}
```

---

## benchmark.json

基準測試模式的輸出。位於 `benchmarks/<timestamp>/benchmark.json`。

```json
{
  "metadata": {
    "skill_name": "pdf",
    "skill_path": "/path/to/pdf",
    "executor_model": "claude-sonnet-4-20250514",
    "analyzer_model": "most-capable-model",
    "timestamp": "2026-01-15T10:30:00Z",
    "evals_run": [1, 2, 3],
    "runs_per_configuration": 3
  },

  "runs": [
    {
      "eval_id": 1,
      "eval_name": "Ocean",
      "configuration": "with_skill",
      "run_number": 1,
      "result": {
        "pass_rate": 0.85,
        "passed": 6,
        "failed": 1,
        "total": 7,
        "time_seconds": 42.5,
        "tokens": 3800,
        "tool_calls": 18,
        "errors": 0
      },
      "expectations": [
        {"text": "...", "passed": true, "evidence": "..."}
      ],
      "notes": [
        "Used 2023 data, may be stale",
        "Fell back to text overlay for non-fillable fields"
      ]
    }
  ],

  "run_summary": {
    "with_skill": {
      "pass_rate": {"mean": 0.85, "stddev": 0.05, "min": 0.80, "max": 0.90},
      "time_seconds": {"mean": 45.0, "stddev": 12.0, "min": 32.0, "max": 58.0},
      "tokens": {"mean": 3800, "stddev": 400, "min": 3200, "max": 4100}
    },
    "without_skill": {
      "pass_rate": {"mean": 0.35, "stddev": 0.08, "min": 0.28, "max": 0.45},
      "time_seconds": {"mean": 32.0, "stddev": 8.0, "min": 24.0, "max": 42.0},
      "tokens": {"mean": 2100, "stddev": 300, "min": 1800, "max": 2500}
    },
    "delta": {
      "pass_rate": "+0.50",
      "time_seconds": "+13.0",
      "tokens": "+1700"
    }
  },

  "notes": [
    "Assertion 'Output is a PDF file' passes 100% in both configurations - may not differentiate skill value",
    "Eval 3 shows high variance (50% ± 40%) - may be flaky or model-dependent",
    "Without-skill runs consistently fail on table extraction expectations",
    "Skill adds 13s average execution time but improves pass rate by 50%"
  ]
}
```

**欄位說明：**
- `metadata`：關於基準測試執行的資訊
  - `skill_name`：skill 的名稱
  - `timestamp`：執行基準測試的時間
  - `evals_run`：eval 名稱或 ID 的清單
  - `runs_per_configuration`：每種設定的執行次數（例如 3）
- `runs[]`：個別執行結果
  - `eval_id`：數字 eval 識別碼
  - `eval_name`：人類可讀的 eval 名稱（用作檢視器的章節標題）
  - `configuration`：必須是 `"with_skill"` 或 `"without_skill"`（檢視器使用此確切字串進行分組與色彩標示）
  - `run_number`：整數執行編號（1、2、3……）
  - `result`：包含 `pass_rate`、`passed`、`total`、`time_seconds`、`tokens`、`errors` 的巢狀物件
- `run_summary`：各設定的統計彙總
  - `with_skill` / `without_skill`：各包含 `pass_rate`、`time_seconds`、`tokens` 物件，其中有 `mean` 與 `stddev` 欄位
  - `delta`：差異字串，例如 `"+0.50"`、`"+13.0"`、`"+1700"`
- `notes`：分析器的自由格式觀察

**重要：** 檢視器會確切讀取這些欄位名稱。若使用 `config` 取代 `configuration`，或將 `pass_rate` 放在執行頂層而非巢狀於 `result` 下，將導致檢視器顯示空白／零值。手動生成 benchmark.json 時，請務必參照此綱要。

---

## comparison.json

盲測比較器的輸出。位於 `<grading-dir>/comparison-N.json`。

```json
{
  "winner": "A",
  "reasoning": "Output A provides a complete solution with proper formatting and all required fields. Output B is missing the date field and has formatting inconsistencies.",
  "rubric": {
    "A": {
      "content": {
        "correctness": 5,
        "completeness": 5,
        "accuracy": 4
      },
      "structure": {
        "organization": 4,
        "formatting": 5,
        "usability": 4
      },
      "content_score": 4.7,
      "structure_score": 4.3,
      "overall_score": 9.0
    },
    "B": {
      "content": {
        "correctness": 3,
        "completeness": 2,
        "accuracy": 3
      },
      "structure": {
        "organization": 3,
        "formatting": 2,
        "usability": 3
      },
      "content_score": 2.7,
      "structure_score": 2.7,
      "overall_score": 5.4
    }
  },
  "output_quality": {
    "A": {
      "score": 9,
      "strengths": ["Complete solution", "Well-formatted", "All fields present"],
      "weaknesses": ["Minor style inconsistency in header"]
    },
    "B": {
      "score": 5,
      "strengths": ["Readable output", "Correct basic structure"],
      "weaknesses": ["Missing date field", "Formatting inconsistencies", "Partial data extraction"]
    }
  },
  "expectation_results": {
    "A": {
      "passed": 4,
      "total": 5,
      "pass_rate": 0.80,
      "details": [
        {"text": "Output includes name", "passed": true}
      ]
    },
    "B": {
      "passed": 3,
      "total": 5,
      "pass_rate": 0.60,
      "details": [
        {"text": "Output includes name", "passed": true}
      ]
    }
  }
}
```

---

## analysis.json

事後分析器的輸出。位於 `<grading-dir>/analysis.json`。

```json
{
  "comparison_summary": {
    "winner": "A",
    "winner_skill": "path/to/winner/skill",
    "loser_skill": "path/to/loser/skill",
    "comparator_reasoning": "Brief summary of why comparator chose winner"
  },
  "winner_strengths": [
    "Clear step-by-step instructions for handling multi-page documents",
    "Included validation script that caught formatting errors"
  ],
  "loser_weaknesses": [
    "Vague instruction 'process the document appropriately' led to inconsistent behavior",
    "No script for validation, agent had to improvise"
  ],
  "instruction_following": {
    "winner": {
      "score": 9,
      "issues": ["Minor: skipped optional logging step"]
    },
    "loser": {
      "score": 6,
      "issues": [
        "Did not use the skill's formatting template",
        "Invented own approach instead of following step 3"
      ]
    }
  },
  "improvement_suggestions": [
    {
      "priority": "high",
      "category": "instructions",
      "suggestion": "Replace 'process the document appropriately' with explicit steps",
      "expected_impact": "Would eliminate ambiguity that caused inconsistent behavior"
    }
  ],
  "transcript_insights": {
    "winner_execution_pattern": "Read skill -> Followed 5-step process -> Used validation script",
    "loser_execution_pattern": "Read skill -> Unclear on approach -> Tried 3 different methods"
  }
}
```