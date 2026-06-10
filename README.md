# Corporate Credit Evaluation Model

This project is for educational and portfolio purposes only. It is not financial advice, investment advice, or an official credit rating system.

Python-based corporate credit scoring model that evaluates public companies using financial statement data, credit ratios, trend metrics, stress testing, SQLite storage, and optional AI-generated credit feedback.

## Overview

This project pulls public company financial data, calculates credit-relevant ratios, assigns internal risk scores, runs downside stress tests, and exports results into CSV and SQLite formats. The model is designed as a finance and credit-analysis project rather than an official rating system.

## Key Features

- Pulls public financial data for a default universe of 20 companies
- Calculates leverage, liquidity, profitability, cash-flow, and coverage ratios
- Assigns internal credit scores and rating categories
- Runs moderate and severe downside stress tests
- Produces rule-based credit feedback
- Optionally generates AI-assisted credit memos using an OpenAI API key
- Saves structured outputs to CSV and SQLite

## How to Run

Install dependencies:

```bash
pip install -r requirements.txt
Run the default model:
python corporate_credit_model_full.py

Run with AI memo generation:
python corporate_credit_model_full.py --ai

Run with custom tickers:
python corporate_credit_model_full.py --tickers AAPL MSFT AMZN NVDA

Known Limitations
The model depends on public financial data availability and consistency.
Internal ratings are model-generated and are not official agency ratings.
Stress scenarios are simplified and should not be treated as full credit underwriting.
AI feedback is optional and should be reviewed critically.

Future Improvements
Add sector-specific scoring weights.
Add visual dashboards for score distribution and stress-test changes.
Expand peer benchmarking by industry.
Add sample output files with anonymized or limited data.
Improve memo formatting for lender-style credit writeups.
