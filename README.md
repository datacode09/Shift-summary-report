# Shift-summary-report
# Operator Conditions Report Generator

**HydroOne / IESO E-log · CPU-only · Local LLM**

Reads the IESO E-log Excel file for a configured shift window and produces a
professional operator handover report using two local LLM models running
entirely on a low-RAM laptop — no cloud, no GPU required.

-----

## Business Requirements

|#    |Requirement                                                                                                              |
|-----|-------------------------------------------------------------------------------------------------------------------------|
|BR-01|The system shall read shift event logs from the IESO E-log Excel file                                                    |
|BR-02|The system shall filter records to a configured shift date and time window                                               |
|BR-03|The system shall enrich every record with the human-readable code definition and priority from `elog_codes.json`         |
|BR-04|The system shall classify every shift record for operational significance using an LLM                                   |
|BR-05|The system shall assign each included record a priority of CRITICAL, HIGH, MEDIUM, or LOW                                |
|BR-06|The system shall detect and flag IESO notifications including “IESO notified”, “IESO informed”, and “Compliance notified”|
|BR-07|Any IESO notification must appear at the very beginning of the final report                                              |
|BR-08|The system shall generate a concise executive handover summary in professional control room language                     |
|BR-09|The report shall open with an overall shift status: “normal”, “eventful”, or “critical”                                  |
|BR-10|CRITICAL and HIGH events shall be listed individually with station or equipment name                                     |
|BR-11|MEDIUM events shall be summarised as a single sentence with a count                                                      |
|BR-12|LOW events shall be acknowledged with a count only                                                                       |
|BR-13|The report shall be 4–6 sentences, plain prose, no bullet points                                                         |
|BR-14|The system shall track token usage and estimated cost per run                                                            |
|BR-15|The system shall run entirely on a low-RAM CPU laptop with no GPU                                                        |
|BR-16|Shift window, file paths, and model settings shall be configurable without code changes                                  |

-----

## Functional Requirements

### Input

|Field              |Description                                                        |
|-------------------|-------------------------------------------------------------------|
|E-log Excel file   |IESO operational log, sheet `OP_LOG_2025_ONWARD`                   |
|`elog_codes.json`  |Code definitions and numeric priorities (1=CRITICAL, 2=HIGH, 3=LOW)|
|`config.json`      |Paths, LLM settings, model configurations                          |
|`shift_config.json`|Shift start hour, end hour, date                                   |

### E-log columns used

|Column         |Usage                                        |
|---------------|---------------------------------------------|
|LOG_ITEM_ID    |Record identifier                            |
|Start Date/Time|Shift window filtering, display timestamp    |
|End Date/Time  |Display timestamp                            |
|Equipment      |Equipment name shown in report               |
|Code           |Looked up in `elog_codes.json` for enrichment|
|Comments       |Primary text fed to LLM for analysis         |
|Compliance     |Completion flag (Y/N)                        |
|SECTOR_NAME    |Sector shown in report                       |

### Processing steps

1. Load `config.json` and `shift_config.json`
1. Load E-log Excel file
1. Filter records to the configured shift window; auto-detect most recent date if today has no records
1. Clean and normalise every record — whitespace, N/A variants, control characters, uppercase codes
1. Enrich every record with `elog_codes.json` — code description and numeric priority injected before LLM sees the record
1. Send every record to `analyze_entry` LLM (Step 1) in batches of `llm_batch_size`
1. Group included records by priority: CRITICAL → HIGH → MEDIUM → LOW
1. Format the `{{entries}}` string for the executive summary
1. Send `{{entries}}` to `executive_summary` LLM (Step 2) — single call per shift
1. Track token usage and estimated cost; write report, log, and audit CSV

### Output — per record (Step 1)

|Field                   |Type       |Description                      |
|------------------------|-----------|---------------------------------|
|`should_include`        |bool       |Whether this record is reportable|
|`summary`               |string     |1–2 sentence operational summary |
|`priority`              |string     |CRITICAL | HIGH | MEDIUM | LOW   |
|`ieso_notified`         |bool       |Whether IESO was informed        |
|`ieso_notification_time`|string|null|HH:MM if notified                |

### Output — files

|File                                |Description                                        |
|------------------------------------|---------------------------------------------------|
|`reports/conditions_report_<ts>.txt`|Operator handover report                           |
|`logs/conditions_<ts>.log`          |Full run log (DEBUG level to file, INFO to console)|
|`logs/audit_<ts>.csv`               |Every record’s classification decision             |

-----

## Non-Functional Requirements

|#     |Requirement                                                                                      |
|------|-------------------------------------------------------------------------------------------------|
|NFR-01|Must run on a CPU-only laptop with no dedicated GPU                                              |
|NFR-02|Total RAM footprint must stay within available system memory — models are unloaded between passes|
|NFR-03|No record may be silently lost — parse failures fall back to individual retry then safe default  |
|NFR-04|ELOG comments are untrusted — the system must not follow instructions embedded in comment text   |
|NFR-05|Raw comment text must be preserved unmodified alongside the cleaned version                      |
|NFR-06|All configuration must be externalisable to JSON files — no operator should need to edit Python  |
|NFR-07|The system must write a full audit trail of every classification decision                        |
|NFR-08|Dev mode must write inspectable intermediate artifacts at every pipeline stage                   |
|NFR-09|Step 2 context window must be computed dynamically from actual prompt size, not hardcoded        |
|NFR-10|Model family (Mistral, Qwen, Llama) must be swappable via config with no code changes            |

-----

## Hardware Requirements

|Component           |Minimum                    |Recommended |
|--------------------|---------------------------|------------|
|RAM                 |8 GB                       |16 GB       |
|CPU cores (physical)|4                          |6–8         |
|Disk (models + data)|10 GB free                 |20 GB free  |
|GPU                 |Not required               |Not required|
|OS                  |Windows 10/11, Linux, macOS|Windows 11  |


> **Important:** Set `n_threads` in `config.json` to your **physical core count**,
> not the logical (hyperthreaded) count. Using hyperthreaded count causes
> cache-line thrashing and significantly degrades performance.

-----

## Software Requirements

### Python dependencies

```
pip install llama-cpp-python pandas openpyxl
```

Python 3.10 or later required (uses `match` syntax and `dataclasses`).

### Model files

**Step 1 — analyze_entry** (classification, JSON output)

- Model: Qwen2.5-3B-Instruct Q4_K_M
- Size: ~1.9 GB
- Download: <https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF>
- File: `qwen2.5-3b-instruct-q4_k_m.gguf`
- Template: `chatml`

**Step 2 — executive_summary** (prose, one call per shift)

- Model: Mistral 7B Instruct v0.2 Q4_K_M
- Size: ~4.1 GB
- Download: <https://huggingface.co/TheBloke/Mistral-7B-Instruct-v0.2-GGUF>
- File: `mistral-7b-instruct-v0.2.Q4_K_M.gguf`
- Template: `mistral`

-----

## Configuration

### Project file layout

```
shift_condition_report/
├── conditions_report_v7.py   ← main script
├── config.json               ← paths and LLM settings
├── shift_config.json         ← shift window
├── elog_codes.json           ← code definitions and priorities
├── Sample_Elog_data.xlsx     ← E-log data
├── qwen2.5-3b-instruct-q4_k_m.gguf
├── mistral-7b-instruct-v0.2.Q4_K_M.gguf
├── reports/                  ← generated reports
├── logs/                     ← run logs and audit CSVs
└── dev/                      ← intermediate artifacts (DEV_MODE only)
```

### config.json

```json
{
  "excel_path":       "./Sample_Elog_data.xlsx",
  "sheet_name":       "OP_LOG_2025_ONWARD",
  "elog_codes_path":  "./elog_codes.json",
  "reports_dir":      "./reports",
  "logs_dir":         "./logs",
  "dev_dir":          "./dev",

  "llm_batch_size":    10,
  "max_comment_chars": 400,
  "dev_mode":          false,
  "n_threads":         4,
  "n_batch":           512,
  "n_gpu_layers":      0,
  "f16_kv":            true,

  "step1_model": {
    "model_path": "./qwen2.5-3b-instruct-q4_k_m.gguf",
    "template":   "chatml",
    "n_ctx":      4096,
    "n_batch":    512,
    "n_threads":  4,
    "f16_kv":     true
  },

  "step2_model": {
    "model_path": "./mistral-7b-instruct-v0.2.Q4_K_M.gguf",
    "template":   "mistral",
    "n_ctx":      4096,
    "n_batch":    512,
    "n_threads":  4,
    "f16_kv":     true
  }
}
```

### shift_config.json

```json
{
  "shift_start_hour": 7,
  "shift_end_hour":   19,
  "shift_date":       null
}
```

Set `shift_date` to `"YYYY-MM-DD"` to pin a specific date.
Leave as `null` to auto-detect the most recent date in the file.

### Key config.json settings

|Key                |Default|Notes                                                          |
|-------------------|-------|---------------------------------------------------------------|
|`llm_batch_size`   |`10`   |Records per LLM prompt. Reduce if context errors occur         |
|`max_comment_chars`|`400`  |Comment truncation before LLM. Reduce to speed up batches      |
|`dev_mode`         |`true` |Set `false` in production — eliminates artifact file writes    |
|`n_threads`        |`4`    |**Must match physical CPU cores, not logical**                 |
|`n_batch`          |`512`  |Prompt prefill chunk size. Reduce to `128` if RAM is very tight|
|`f16_kv`           |`true` |Halves KV cache memory. Keep `true` on low-RAM machines        |

### Supported model templates

|`template` value|Model families                   |
|----------------|---------------------------------|
|`"mistral"`     |Mistral 7B, Mistral Small        |
|`"chatml"`      |Qwen2.5 (all sizes), Phi-3, Phi-4|
|`"llama3"`      |Llama 3.2 1B/3B, Llama 3.3 8B    |

-----

## Running

```bash
python conditions_report_v7.py
```

The script prints INFO-level progress to the console and writes full DEBUG
logs to `logs/conditions_<timestamp>.log`.

-----

## Dev Mode

Set `"dev_mode": true` in `config.json` to write inspectable intermediate
files to `dev/<run_timestamp>/` for every pipeline stage:

|File                           |Content                                            |
|-------------------------------|---------------------------------------------------|
|`00_raw_shift_rows.csv`        |Exact rows extracted from Excel after shift filter |
|`01_prepared_enriched_rows.csv`|After cleaning and `elog_codes.json` enrichment    |
|`02_batch_NNNN_prompt.txt`     |Exact prompt sent to LLM for each batch            |
|`02_batch_NNNN_response.txt`   |Raw LLM output for each batch                      |
|`02_batch_NNNN_parsed.json`    |Parsed JSON result per batch                       |
|`03_after_analyze_entry.csv`   |All records with LLM classification decisions      |
|`04_entries_string.txt`        |The `{{entries}}` variable fed to executive summary|
|`05_summary_prompt.txt`        |Exact executive summary prompt                     |
|`05_summary_response.txt`      |Raw executive summary output                       |

Use dev mode when tuning prompts or diagnosing classification decisions.
Set `"dev_mode": false` before deploying — it has zero overhead when off.

-----

## elog_codes.json Priority Scale

|Numeric priority|Label   |Meaning                                             |
|----------------|--------|----------------------------------------------------|
|`1`             |CRITICAL|Forced outages, fires, safety events — always report|
|`2`             |HIGH    |Extended outages, significant events                |
|`3`             |LOW     |Routine, informational                              |

The LLM receives both the numeric priority and the code description for
every record before making its classification decision.

-----

## Troubleshooting

**`"Parse error — manual review"` in audit CSV**
The LLM output could not be parsed as JSON. Check `02_batch_NNNN_response.txt`
in the dev folder. Common causes: context window too small (reduce
`llm_batch_size`) or model echoing the prompt (already handled by
`strip_prompt_echo()` but may vary by llama-cpp version).

**`Summary prompt ~XXXX tok — capping at 150`**
Too many included records to fit in the executive summary context. The system
will automatically use hierarchical summarisation — HIGH records are grouped
into mini-summaries first, then compressed entries are fed to the final summary.
If this happens regularly, consider reducing `llm_batch_size` or `max_comment_chars`.

**`0 rows matched — using last 200 rows`**
The configured shift window has no records for today. The system auto-selects
the most recent date in the file. Set `shift_date` in `shift_config.json` to
pin a specific date.

**Out of memory during model load**
Reduce `n_batch` from `512` to `128` in `config.json`. This reduces the RAM
spike during prompt evaluation with minimal speed impact.
