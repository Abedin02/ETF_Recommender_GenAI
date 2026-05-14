# ETF Recommender System with GenAI

This project is a data warehousing and recommendation-system proof of concept for finding ETF alternatives with similar holdings and investment strategies. The notebook builds an ETF universe, collects holdings data, stores the data in MongoDB, generates AI strategy summaries, validates data quality, and recommends similar ETFs using multiple similarity metrics.

Main notebook:

- `ETF_Recommender_System_with_GenAI.ipynb`

Public-sharing status:

- The notebook does not include hardcoded MongoDB credentials or Gemini API keys.
- Notebook outputs, execution counts, and widget metadata have been cleared.
- Secrets must be provided through environment variables or Colab Secrets.
- `.gitignore` excludes local secret files such as `.env`, private key files, notebook checkpoints, logs, and temporary files.

## Project Overview

The system answers a practical investment question: given an ETF ticker such as `SPY`, `QQQ`, or `ARKK`, what other ETFs provide similar exposure?

The project combines:

- ETF metadata scraping and cleaning
- ETF holdings collection from provider or market-data sources
- MongoDB storage with date-based warehouse design
- Indexing and query-performance analysis
- LLM-generated ETF strategy summaries
- Data quality checks for holdings and summaries
- ETF similarity recommendations using holdings and semantic summaries

## Contributors

| Contributor | Project Parts | Responsibilities | Email | GitHub / Repo |
| --- | --- | --- | --- | --- |
| Awab | Part 1 | ETF master collection, Finviz scraping, ETF metadata cleaning, AUM parsing, duplicate removal, query date creation, and initial index-performance comparison. | abedinawab1@gmail.com |
| Tanat | Part 2 | ETF holdings collection, iShares scraping, provider URL extraction, holdings CSV retrieval, Selenium setup, StockAnalysis fallback exploration, and resume-capability setup. | parsajafaripour@gmail.com | 
| Pratham | Parts 3 and 4 | MongoDB connection, database and collection setup, bulk upserts, index creation, explain-plan performance comparisons, Gemini integration, prompt creation, self-healing strategy-summary generation, sample output review, and hallucination checks. | shahpratham38@gmail.com |
| Parsa | Parts 5 and 6 | Data quality validation, holdings cleanup, ticker normalization, weight parsing, similarity metric implementation, embedding generation, known-pair validation, and ETF recommendation output. | tsahta1@pride.hofstra.edu | 
## Architecture

The project uses two MongoDB collections:

### `etf_master`

Stores ETF-level metadata such as:

- `etf_ticker`
- `fund_name`
- `asset_class`
- `aum`
- `expense_ratio`
- `query_date`
- other source-specific fields

The `query_date` field tracks when the data was collected so daily ETF snapshots can be stored and compared over time.

### `etf_holdings`

Stores holdings-level data per ETF:

```python
{
    "etf_ticker": "SPY",
    "query_date": "YYYYMMDD",
    "source": "ishares",
    "holdings": [
        {"stock_ticker": "AAPL", "weight": 7.1},
        {"stock_ticker": "MSFT", "weight": 6.5}
    ],
    "strategy_summary": "Plain-English LLM summary of the ETF strategy."
}
```

## Project Parts

### Part 1: ETF Master Collection

Implemented by Awab.

This section scrapes ETF metadata, cleans the resulting table, converts numeric fields, removes duplicate ETF tickers, adds a daily `query_date`, and compares MongoDB query performance before and after indexing.

Key techniques:

- `requests`
- `BeautifulSoup`
- `pandas`
- rotating user agents
- retry handling
- AUM and percentage parsing
- MongoDB `explain` plans

### Part 2: ETF Holdings Collection

Implemented by Tanat.

This section collects ETF holdings data, starting with iShares product pages and holdings CSV links. It also includes Selenium setup and StockAnalysis exploration as an alternate data source. Resume capability is included so existing local data can be loaded instead of restarting the collection process.

Key techniques:

- iShares product screener scraping
- holdings CSV construction
- Selenium / ChromeDriver setup
- fallback source exploration
- resumable data collection

### Part 3: MongoDB Implementation

Implemented by Pratham.

This section connects to MongoDB, creates the `etf_project` database, defines the `etf_master` and `etf_holdings` collections, inserts or upserts data, and creates indexes for common query patterns.

Important indexes:

```python
etf_master_col.create_index([("query_date", -1), ("etf_ticker", 1)], unique=True)
etf_master_col.create_index("etf_ticker")
etf_master_col.create_index("aum")

etf_holdings_col.create_index([("query_date", -1), ("etf_ticker", 1)], unique=True)
etf_holdings_col.create_index("etf_ticker")
etf_holdings_col.create_index("holdings.stock_ticker")
```

The notebook uses MongoDB `explain` output to compare collection scans with indexed queries.

### Part 4: AI-Generated Strategy Summaries

Implemented by Pratham.

This section uses Google Gemini to generate plain-English strategy summaries from the top holdings of each ETF. The summaries are stored in MongoDB under `strategy_summary`.

The summary-generation pipeline is designed to be self-healing:

- It finds holdings documents without summaries.
- It generates summaries in batches.
- It stops gracefully when rate limits occur.
- It can be rerun later and resumes from missing summaries.

### Part 5: Data Quality Validation

Implemented by Parsa.

This section validates and cleans holdings data before similarity calculations. It normalizes stock tickers, parses weight values, merges duplicate holdings, scales decimal weights when needed, checks whether weights sum to roughly 100%, and reports issue counts.

Validation checks include:

- holdings field is a list
- valid stock tickers
- numeric positive weights
- reasonable total weight
- presence of strategy summaries

### Part 6: ETF Recommendations

Implemented by Parsa.

This section recommends similar ETFs using three metrics:

1. **Jaccard similarity** on holdings membership
2. **Cosine similarity** on holdings weights
3. **Cosine similarity** on strategy-summary embeddings

The notebook builds sentence embeddings from ETF strategy summaries using `sentence-transformers`, compares ETFs, validates known similar pairs such as `SPY` / `VOO` and `QQQ` / `XLK`, and displays top recommendations for example tickers.

## Requirements

The notebook installs several packages as needed, including:

- `requests`
- `beautifulsoup4`
- `pandas`
- `numpy`
- `pymongo`
- `certifi`
- `random-user-agent`
- `selenium`
- `webdriver-manager`
- `google-generativeai`
- `sentence-transformers`
- `scikit-learn`

If running locally, install the core dependencies with:

```bash
pip install requests beautifulsoup4 pandas numpy pymongo certifi random-user-agent selenium webdriver-manager google-generativeai sentence-transformers scikit-learn
```

Some notebook cells use Linux/Colab shell commands such as `apt-get`, `wget`, and Chrome installation commands. Those cells are intended for Google Colab or a compatible Linux environment.

## Environment Variables

This project requires secrets to be provided through environment variables or Colab Secrets. The notebook intentionally raises an error if either value is missing.

```bash
MONGODB_URI="your_mongodb_connection_string"
GEMINI_API_KEY="your_gemini_api_key"
```

Do not commit database credentials or API keys to GitHub. If any keys were used during experimentation or appeared in an older notebook version, rotate them before publishing the repository.

Example Python access pattern used in the notebook:

```python
URI = os.getenv("MONGODB_URI")
if not URI:
    raise ValueError("MONGODB_URI is not set. Add it as an environment variable or Colab Secret before running this notebook.")

GEMINI_API_KEY = os.getenv("GEMINI_API_KEY")
if not GEMINI_API_KEY:
    raise ValueError("GEMINI_API_KEY is not set. Add it as an environment variable or Colab Secret before running this notebook.")
```

## How to Run

1. Open `ETF_Recommender_System_with_GenAI.ipynb` in Google Colab or Jupyter.
2. Set `MONGODB_URI` and `GEMINI_API_KEY` in the runtime environment.
3. Run Part 1 to create and clean the ETF master dataset.
4. Run Part 2 to collect ETF holdings.
5. Run Part 3 to connect to MongoDB, insert data, and create indexes.
6. Run Part 4 to generate missing ETF strategy summaries.
7. Run Part 5 to validate and clean holdings data.
8. Run Part 6 to generate recommendations and compare similarity metrics.

## Example Usage

The recommendation section runs test queries for:

```python
test_queries = ["SPY", "QQQ", "ARKK"]
```

For each ETF, the notebook returns:

- candidate ETF ticker
- Jaccard similarity
- holdings cosine similarity
- summary embedding similarity
- combined score
- shared holdings count
- top shared holdings
- candidate strategy summary

## AI Disclosure

The notebook uses Google Gemini through the `google-generativeai` package to generate ETF strategy summaries. The prompt asks the model to act as a financial analyst and summarize an ETF based on its top holdings.

The project also includes sample generated summaries and a hallucination-check section that compares summary claims against actual holdings.

Before final submission, document:

- exact LLM used
- final prompt template
- 3 to 5 generated summary examples
- any hallucinations found
- how hallucinations were checked or handled

## Public Sharing Checklist

Before publishing or submitting this repository:

- Confirm `MONGODB_URI` and `GEMINI_API_KEY` are stored only in your local environment, Colab Secrets, or a secure deployment environment.
- Run a secret scan for patterns such as `mongodb+srv`, API key prefixes, passwords, tokens, and private keys.
- Keep notebook outputs cleared unless the assignment specifically requires executed output.
- Do not upload `.env`, `.pem`, `.key`, `.crt`, temporary CSVs, or notebook checkpoint folders.
- Rotate any key that was previously pasted into the notebook or shared outside the team.

## Notes

- The project is designed around daily snapshots, so `query_date` is required for both master and holdings collections.
- MongoDB indexes are important because the project queries by `query_date`, `etf_ticker`, `aum`, and nested `holdings.stock_ticker`.
- Recommendation quality depends heavily on clean stock tickers, numeric weights, and complete strategy summaries.
- Some web sources may block automated requests or change HTML structure, so scraping cells may need minor updates over time.
