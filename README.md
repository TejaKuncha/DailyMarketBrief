# Daily Market Brief

A UiPath automation that pulls daily OHLCV data for a configurable list of stock
tickers from the [Twelve Data](https://twelvedata.com/) API, asks a Groq-hosted
LLM to summarize the day's action, writes the results to a dated sheet in an
Excel workbook, and emails a formatted HTML brief highlighting the day's
significant movers.

## What it does

1. Loads run configuration from `Data\Config.xlsx` (`Settings`, `Constants`,
   `Assets` sheets) into a single key/value dictionary.
2. Retrieves the Twelve Data and Groq API keys from Orchestrator Assets.
3. Ensures the output workbook (`DailyMarketBrief.xlsx`) exists, copying it
   from the template if this is the first run.
4. For each configured ticker: calls the Twelve Data Time Series API for
   open/close/volume, caches the raw response to `Data\{TICKER}.txt` for the
   day (so re-running the process doesn't re-hit the API), and computes the
   percent change vs. the prior close.
5. Sends the collected ticker data to the Groq Chat Completions API and gets
   back a short natural-language market brief.
6. Duplicates the template sheet into a new sheet named for today's date,
   writes the brief and ticker table into it, and color-highlights tickers
   that moved beyond the configured threshold.
7. Builds an HTML email (brief + significant movers + full ticker table) and
   sends it via Gmail.
8. On any unhandled error, sends a failure notification email and terminates
   the run; each step also logs progress/errors via `Log Message`.

The full logic lives in [`Main.xaml`](Main.xaml).

## Prerequisites

- UiPath Studio (Windows platform target), with these dependencies restored
  (see [`project.json`](project.json)):
  - `UiPath.Excel.Activities`
  - `UiPath.GSuite.Activities`
  - `UiPath.System.Activities`
  - `UiPath.WebAPI.Activities`
- A UiPath Orchestrator connection with two Credential/Text assets:
  - Twelve Data API key
  - Groq API key
- A configured Gmail connection (used by the `Send Email` activities).
- A [Twelve Data](https://twelvedata.com/) account/API key and a
  [Groq](https://groq.com/) account/API key.

## Configuration

All runtime settings are read from `Data\Config.xlsx`. Every sheet
(`Settings`, `Constants`, `Assets`) is expected to have `Name` / `Value`
columns and is merged into one lookup dictionary, so the workflow doesn't
care which sheet a given key lives on. Keys currently read by `Main.xaml`:

| Key | Purpose |
|---|---|
| `Orchestrator Folder` | Orchestrator folder containing the API key assets |
| `API Key_TwelveData` | Asset name for the Twelve Data API key |
| `API Key_Groq` | Asset name for the Groq API key |
| `Output File Path` | Folder for `DailyMarketBrief.xlsx` (defaults to Desktop if blank) |
| `Template File path` | Path to the template workbook copied on first run |
| `Template Sheetname` | Sheet name duplicated for each day's report |
| `Stock Tickers` | Comma-separated list of tickers to process |
| `Groq_timeseries_API` | Twelve Data Time Series request URL format string |
| `JSON Path_open` / `JSON Path_close_prevDay` / `JSON Path_close_twoDays` / `JSON Path_volume` | JSONPath expressions into the Twelve Data response |
| `Groq LLM API URL` | Groq Chat Completions endpoint |
| `Groq System Prompt` | System prompt sent to the LLM |
| `Groq Model Temperature` / `Groq Model Max Tokens` / `Groq LLM Model` | Groq request parameters |
| `JSON Path_groqResponse` | JSONPath into the Groq response for the brief text |
| `Daily Brief Header Cell` / `Daily Brief Groq Response Cell` / `Daily Brief Ticker Table Range` | Cell/range addresses used when writing the output sheet |
| `Percentage Change Threshold` | % change (absolute value) that counts as a "significant mover" |
| `Email To` | Semicolon-separated recipient list |

## Project structure

```
Main.xaml                        Entry-point workflow (all logic)
project.json                     UiPath project manifest / dependencies
Data/Config.xlsx                 Run configuration (Settings/Constants/Assets)
Data/DailyMarketBrief_template.xlsx  Template sheet copied for each day's report
Data/*.txt                       Per-ticker cached API responses (generated, gitignored)
```

## Running

Open the project in UiPath Studio and run `Main.xaml`, or publish it and run
it from Orchestrator as an unattended process. On success, a new dated sheet
is added to `DailyMarketBrief.xlsx` and a summary email is sent; on failure,
a failure email is sent and the run terminates.
