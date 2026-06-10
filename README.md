# Corporate Credit Evaluation Model

## Purpose

I built this project to practice corporate credit analysis using Python. The model analyzes public-company financial data, calculates credit-relevant ratios, applies simple stress scenarios, and organizes the results for review.

The main goal was to connect financial statement analysis with a repeatable coding workflow. I wanted to better understand how leverage, liquidity, profitability, and coverage ratios can be used together when evaluating corporate credit risk.

## What It Does

The model is designed to evaluate a group of public companies using a structured credit-analysis process.

Core features include:

- Pulling or processing public financial data
- Calculating credit-relevant financial ratios
- Comparing companies across a common framework
- Applying basic stress-case assumptions
- Producing structured output for review
- Saving results in CSV and/or SQLite format
- Optional AI-assisted credit summary generation

## Credit Metrics

The model may evaluate companies using metrics such as:

- Revenue
- EBITDA or operating income
- Net income
- Total debt
- Cash and equivalents
- Debt-to-equity
- Debt-to-assets
- Interest coverage
- Current ratio
- Profit margin
- Return on assets
- Stress-case coverage estimates

These metrics are intended to provide a high-level view of credit quality, not a full professional credit rating.

## Project Structure

```text
corporate-credit-evaluation-model/
│
├── corporate_credit_model.py
├── requirements.txt
├── .gitignore
├── .env.example
├── LICENSE
└── README.md
```

## How to Run

Clone the repository:

```bash
git clone https://github.com/williamtbon/corporate-credit-evaluation-model.git
cd corporate-credit-evaluation-model
```

Install dependencies:

```bash
pip install -r requirements.txt
```

Run the model:

```bash
python corporate_credit_model.py
```

## Configuration

If the model uses APIs or optional AI-assisted summaries, store keys locally in a `.env` file.

Example:

```text
DATA_API_KEY=your_data_api_key_here
OPENAI_API_KEY=your_optional_key_here
```

Do not upload real API keys or credentials to GitHub.

## Example Workflow

A typical workflow is:

1. Select a group of companies to analyze.
2. Pull or load financial data.
3. Calculate credit ratios.
4. Apply stress assumptions.
5. Compare results across firms.
6. Export the output for review.
7. Optionally generate a plain-English credit summary.

## What I Learned

While building this project, I learned that:

- Credit analysis should not rely on one ratio alone.
- Leverage, liquidity, profitability, and coverage need to be reviewed together.
- Public financial data can be inconsistent, so cleaning and validation are important.
- Stress testing helps show how quickly a company’s credit profile can weaken under pressure.
- AI-generated summaries can be useful, but the underlying ratio logic needs to remain transparent and reviewable.

## Limitations

This project has several limitations:

- It is a simplified credit model and not a formal credit-rating system.
- Public data may be delayed, incomplete, or inconsistent.
- The model does not fully replace manual credit judgment.
- Industry-specific differences may require more tailored ratio interpretation.
- Stress assumptions are simplified and may not reflect real market conditions.

## Future Improvements

Possible future improvements include:

- Adding industry-specific credit scoring logic
- Improving data validation checks
- Adding charts or dashboards
- Expanding stress-testing scenarios
- Adding a formal credit memo output
- Separating the code into modules for data, ratios, scoring, and reporting
- Adding unit tests for ratio calculations

## Disclaimer

This project is for educational and portfolio purposes only. It is not financial advice, investment advice, credit advice, or a professional credit rating.
