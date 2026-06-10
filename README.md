# Corporate Credit Evaluation Model

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
