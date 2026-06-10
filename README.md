# Corporate Credit Evaluation Model

## Purpose

I built this project to practice corporate credit analysis with Python. The goal was to create a repeatable workflow for reviewing public-company financial data, calculating credit-related ratios, applying basic stress assumptions, and organizing the results.

I wanted this project to connect finance concepts with actual code. Instead of only calculating ratios manually, I wanted to build something that could process multiple companies in a consistent way and produce outputs that are easier to review.

## What This Project Does

This model is designed to evaluate selected public companies through a basic corporate credit framework.

Main features include:

* Pulling or processing public financial data
* Calculating credit-related financial ratios
* Reviewing leverage, liquidity, profitability, and coverage
* Applying simple stress-case assumptions
* Comparing companies across the same framework
* Saving outputs in CSV and/or SQLite format
* Producing an optional plain-English summary of results

The project is not meant to replace professional credit judgment. It is a learning tool for practicing how credit analysis can be structured with code.

## Credit Metrics

The model may evaluate companies using metrics such as:

* Revenue
* Operating income or EBITDA
* Net income
* Total debt
* Cash and equivalents
* Debt-to-equity
* Debt-to-assets
* Interest coverage
* Current ratio
* Profit margin
* Return on assets
* Stress-case coverage estimates

These ratios are meant to provide a high-level view of credit quality. They should be interpreted carefully because companies can differ significantly by industry, capital structure, and reporting style.

## Project Structure

```text
corporate-credit-evaluation-model/
│
├── corporate_credit_model.py
├── requirements.txt
├── .env.example
├── .gitignore
├── LICENSE
└── README.md
```

## How to Run

Clone the repository:

```bash
git clone https://github.com/williamtbon/corporate-credit-evaluation-model.git
cd corporate-credit-evaluation-model
```

Install the required packages:

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

Do not upload real API keys, credentials, or private data to GitHub.

## Example Workflow

A typical workflow for this project is:

1. Select a group of companies.
2. Pull or load public financial data.
3. Calculate credit ratios.
4. Apply basic stress assumptions.
5. Compare results across firms.
6. Export the results for review.
7. Optionally generate a plain-English summary.

## Notes From Building This

This project helped me better understand that credit analysis is not based on one perfect number. A company can look strong on one metric and weaker on another, so leverage, liquidity, profitability, and coverage need to be reviewed together.

One challenge was making the output useful without pretending the model is more advanced than it is. The project is intentionally simplified, but it gave me practice building a structured credit workflow and thinking about how financial data can be turned into a repeatable analysis process.

## Limitations

This project has several limitations:

* It is a simplified credit model, not a formal credit-rating system.
* Public financial data may be incomplete, delayed, or inconsistent.
* The model does not fully adjust for industry differences.
* Stress assumptions are basic and may not reflect real market conditions.
* The output should be reviewed manually before drawing conclusions.
* AI-assisted summaries, if used, should be treated as explanations of the model output, not independent credit opinions.

## Future Improvements

Future improvements could include:

* Adding industry-specific ratio interpretation
* Improving data validation
* Expanding stress-testing scenarios
* Adding charts or dashboards
* Creating a more formal credit memo output
* Separating the code into modules for data, ratios, scoring, and reporting
* Adding unit tests for ratio calculations

## Disclaimer

This project is for educational and portfolio purposes only. It is not financial advice, investment advice, credit advice, or a professional credit rating.
