"""
Corporate Credit Evaluation Model
Full single-file version: 20 firms + caching + trend metrics + stress testing + AI feedback + SQLite.

Run:
    python corporate_credit_model_full.py

Run with one AI memo:
    python corporate_credit_model_full.py --ai

Run with refreshed data:
    python corporate_credit_model_full.py --refresh

Run with custom tickers:
    python corporate_credit_model_full.py --tickers AAPL MSFT AMZN NVDA

Install:
    pip install yfinance pandas numpy openai

Outputs:
    outputs/corporate_credit_results.csv
    outputs/stress_test_results.csv
    outputs/raw_borrowers.csv
    outputs/rule_based_credit_feedback.txt
    outputs/ai_credit_feedback.txt
    outputs/corporate_credit_model.db

AI setup:
    set OPENAI_API_KEY=your_api_key
    optional: set OPENAI_MODEL=gpt-5.4-mini
    optional: set OPENAI_MAX_OUTPUT_TOKENS=900
"""

from __future__ import annotations

import argparse
import json
import os
import sqlite3
import sys
import time
import warnings
from dataclasses import asdict, dataclass, field
from datetime import datetime, timedelta
from pathlib import Path
from typing import Any, Dict, List, Optional, Tuple

import numpy as np
import pandas as pd
import yfinance as yf

warnings.filterwarnings("ignore")

DEFAULT_TICKERS = [
    "AAPL", "MSFT", "AMZN", "GOOGL", "META", "NVDA", "TSLA", "WMT", "COST", "HD",
    "MCD", "KO", "PEP", "NKE", "DIS", "XOM", "CVX", "CAT", "BA", "GE"
]

SECTOR_RISK_MAP = {
    "Technology": 2.5,
    "Communication Services": 3.0,
    "Consumer Cyclical": 3.5,
    "Consumer Defensive": 2.0,
    "Healthcare": 2.5,
    "Industrials": 3.0,
    "Energy": 4.0,
    "Basic Materials": 3.5,
    "Utilities": 2.0,
    "Real Estate": 3.5,
    "Financial Services": 4.0,
    "Unknown": 3.0,
}


def safe_div(numerator: Optional[float], denominator: Optional[float]) -> Optional[float]:
    try:
        if numerator is None or denominator is None:
            return None
        if pd.isna(numerator) or pd.isna(denominator) or denominator == 0:
            return None
        return float(numerator) / float(denominator)
    except Exception:
        return None


def clean_number(value: Any) -> Optional[float]:
    try:
        if value is None or pd.isna(value):
            return None
        return float(value)
    except Exception:
        return None


def clamp(value: float, low: float = 0.0, high: float = 100.0) -> float:
    return max(low, min(value, high))


def latest(values: List[Optional[float]]) -> Optional[float]:
    for value in values:
        if value is not None and not pd.isna(value):
            return value
    return None


def compute_cagr(values: List[Optional[float]]) -> Optional[float]:
    clean = [x for x in values if x is not None and x > 0]
    if len(clean) < 2:
        return None
    start = clean[-1]
    end = clean[0]
    periods = len(clean) - 1
    if start <= 0 or periods <= 0:
        return None
    try:
        return (end / start) ** (1 / periods) - 1
    except Exception:
        return None


def ensure_dir(path: Path) -> None:
    path.mkdir(parents=True, exist_ok=True)


def now_iso() -> str:
    return datetime.utcnow().replace(microsecond=0).isoformat()


def normalize_tickers(tickers: List[str]) -> List[str]:
    out = []
    for ticker in tickers:
        t = ticker.strip().upper()
        if t and t not in out:
            out.append(t)
    return out


def get_statement(ticker_obj: yf.Ticker, statement_name: str) -> pd.DataFrame:
    try:
        statement = getattr(ticker_obj, statement_name)
        return statement if statement is not None else pd.DataFrame()
    except Exception:
        return pd.DataFrame()


def get_history(
    statement: pd.DataFrame,
    possible_names: List[str],
    max_periods: int = 4,
    absolute: bool = False,
) -> List[Optional[float]]:
    if statement is None or statement.empty:
        return []
    for name in possible_names:
        if name in statement.index:
            row = statement.loc[name].dropna()
            values = []
            for raw_value in row.iloc[:max_periods].tolist():
                value = clean_number(raw_value)
                if value is not None and absolute:
                    value = abs(value)
                values.append(value)
            return values
    return []


def sum_histories(histories: List[List[Optional[float]]], max_periods: int = 4) -> List[Optional[float]]:
    out = []
    for i in range(max_periods):
        total = 0.0
        found = False
        for history in histories:
            if i < len(history) and history[i] is not None:
                total += float(history[i])
                found = True
        out.append(total if found else None)
    while out and out[-1] is None:
        out.pop()
    return out


@dataclass
class Borrower:
    ticker: str
    company_name: str
    sector: str
    industry: str
    revenue: Optional[float]
    ebitda: Optional[float]
    ebit: Optional[float]
    interest_expense: Optional[float]
    total_debt: Optional[float]
    cash: Optional[float]
    total_assets: Optional[float]
    total_equity: Optional[float]
    current_assets: Optional[float]
    current_liabilities: Optional[float]
    inventory: Optional[float]
    operating_cash_flow: Optional[float]
    capex: Optional[float]
    history: Dict[str, List[Optional[float]]] = field(default_factory=dict)
    sector_risk_score: float = 3.0
    management_score: float = 3.0
    customer_concentration_score: float = 3.0


def sector_risk(sector: str) -> float:
    return SECTOR_RISK_MAP.get(sector, 3.0)


def extract_borrower_from_yfinance(ticker_symbol: str) -> Borrower:
    ticker_symbol = ticker_symbol.upper()
    t = yf.Ticker(ticker_symbol)

    try:
        info = t.info or {}
    except Exception:
        info = {}

    company_name = info.get("longName") or info.get("shortName") or ticker_symbol
    sector = info.get("sector") or "Unknown"
    industry = info.get("industry") or "Unknown"

    income = get_statement(t, "income_stmt")
    balance = get_statement(t, "balance_sheet")
    cashflow = get_statement(t, "cashflow")

    revenue = get_history(income, ["Total Revenue", "Operating Revenue", "Revenue"])
    ebitda = get_history(income, ["EBITDA", "Normalized EBITDA"])
    ebit = get_history(income, ["EBIT", "Operating Income"])
    interest = get_history(income, ["Interest Expense", "Interest Expense Non Operating"], absolute=True)

    debt = get_history(balance, ["Total Debt"])
    if not debt:
        ltd = get_history(balance, ["Long Term Debt", "Long Term Debt And Capital Lease Obligation"])
        curd = get_history(balance, ["Current Debt", "Short Long Term Debt", "Current Debt And Capital Lease Obligation"])
        debt = sum_histories([ltd, curd])

    cash = get_history(balance, ["Cash And Cash Equivalents", "Cash Cash Equivalents And Short Term Investments", "Cash Financial"])
    assets = get_history(balance, ["Total Assets"])
    equity = get_history(balance, ["Stockholders Equity", "Common Stock Equity", "Total Equity Gross Minority Interest"])
    current_assets = get_history(balance, ["Current Assets", "Total Current Assets"])
    current_liabilities = get_history(balance, ["Current Liabilities", "Total Current Liabilities"])
    inventory = get_history(balance, ["Inventory", "Inventories"])
    ocf = get_history(cashflow, ["Operating Cash Flow", "Cash Flow From Continuing Operating Activities", "Total Cash From Operating Activities"])
    capex = get_history(cashflow, ["Capital Expenditure", "Capital Expenditures"], absolute=True)

    history = {
        "revenue": revenue,
        "ebitda": ebitda,
        "ebit": ebit,
        "interest_expense": interest,
        "total_debt": debt,
        "cash": cash,
        "total_assets": assets,
        "total_equity": equity,
        "current_assets": current_assets,
        "current_liabilities": current_liabilities,
        "inventory": inventory,
        "operating_cash_flow": ocf,
        "capex": capex,
    }

    return Borrower(
        ticker=ticker_symbol,
        company_name=company_name,
        sector=sector,
        industry=industry,
        revenue=latest(revenue),
        ebitda=latest(ebitda),
        ebit=latest(ebit),
        interest_expense=latest(interest),
        total_debt=latest(debt),
        cash=latest(cash),
        total_assets=latest(assets),
        total_equity=latest(equity),
        current_assets=latest(current_assets),
        current_liabilities=latest(current_liabilities),
        inventory=latest(inventory),
        operating_cash_flow=latest(ocf),
        capex=latest(capex),
        history=history,
        sector_risk_score=sector_risk(sector),
    )


def cache_valid(path: Path, cache_days: int) -> bool:
    if not path.exists():
        return False
    return datetime.now() - datetime.fromtimestamp(path.stat().st_mtime) <= timedelta(days=cache_days)


def load_cache(path: Path) -> Borrower:
    with open(path, "r", encoding="utf-8") as f:
        return Borrower(**json.load(f))


def save_cache(borrower: Borrower, path: Path) -> None:
    ensure_dir(path.parent)
    with open(path, "w", encoding="utf-8") as f:
        json.dump(asdict(borrower), f, indent=2)


def pull_or_load_borrower(ticker: str, cache_dir: Path, cache_days: int, refresh: bool, sleep_seconds: float) -> Optional[Borrower]:
    ticker = ticker.upper()
    path = cache_dir / f"{ticker}.json"
    if not refresh and cache_valid(path, cache_days):
        print(f"Using cached data for {ticker}")
        try:
            return load_cache(path)
        except Exception as exc:
            print(f"Cache failed for {ticker}: {exc}")

    print(f"Pulling data for {ticker}")
    try:
        borrower = extract_borrower_from_yfinance(ticker)
        save_cache(borrower, path)
        if sleep_seconds > 0:
            time.sleep(sleep_seconds)
        return borrower
    except Exception as exc:
        print(f"Failed to pull {ticker}: {exc}")
        return None


class CorporateCreditModel:
    MAX_POINTS = {
        "debt_to_ebitda": 20,
        "interest_coverage": 18,
        "fcf_to_debt": 15,
        "current_ratio": 8,
        "quick_ratio": 6,
        "ebitda_margin": 11,
        "debt_to_capital": 10,
        "cash_to_debt": 6,
        "asset_coverage": 6,
    }

    def calculate_ratios(self, b: Borrower) -> Dict[str, Optional[float]]:
        fcf = b.operating_cash_flow - b.capex if b.operating_cash_flow is not None and b.capex is not None else None
        net_debt = b.total_debt - b.cash if b.total_debt is not None and b.cash is not None else None
        quick_assets = b.current_assets - b.inventory if b.current_assets is not None and b.inventory is not None else b.current_assets
        debt_plus_equity = b.total_debt + b.total_equity if b.total_debt is not None and b.total_equity is not None else None
        return {
            "debt_to_ebitda": safe_div(b.total_debt, b.ebitda),
            "net_debt_to_ebitda": safe_div(net_debt, b.ebitda),
            "interest_coverage": safe_div(b.ebit, b.interest_expense),
            "ebitda_interest_coverage": safe_div(b.ebitda, b.interest_expense),
            "fcf_to_debt": safe_div(fcf, b.total_debt),
            "current_ratio": safe_div(b.current_assets, b.current_liabilities),
            "quick_ratio": safe_div(quick_assets, b.current_liabilities),
            "ebitda_margin": safe_div(b.ebitda, b.revenue),
            "debt_to_capital": safe_div(b.total_debt, debt_plus_equity),
            "cash_to_debt": safe_div(b.cash, b.total_debt),
            "asset_coverage": safe_div(b.total_assets, b.total_debt),
            "free_cash_flow": fcf,
            "net_debt": net_debt,
        }

    def calculate_trends(self, b: Borrower) -> Dict[str, Optional[float]]:
        h = b.history
        revenue_cagr = compute_cagr(h.get("revenue", []))
        ebitda_cagr = compute_cagr(h.get("ebitda", []))

        dte_hist = []
        debt = h.get("total_debt", [])
        ebitda = h.get("ebitda", [])
        for i in range(max(len(debt), len(ebitda))):
            dte_hist.append(safe_div(debt[i] if i < len(debt) else None, ebitda[i] if i < len(ebitda) else None))
        dte_clean = [x for x in dte_hist if x is not None]
        dte_change = dte_clean[0] - dte_clean[-1] if len(dte_clean) >= 2 else None

        ic_hist = []
        ebit = h.get("ebit", [])
        interest = h.get("interest_expense", [])
        for i in range(max(len(ebit), len(interest))):
            ic_hist.append(safe_div(ebit[i] if i < len(ebit) else None, interest[i] if i < len(interest) else None))
        ic_clean = [x for x in ic_hist if x is not None]
        ic_change = ic_clean[0] - ic_clean[-1] if len(ic_clean) >= 2 else None

        fcf_hist = []
        ocf = h.get("operating_cash_flow", [])
        capex = h.get("capex", [])
        for i in range(max(len(ocf), len(capex))):
            if i < len(ocf) and i < len(capex) and ocf[i] is not None and capex[i] is not None:
                fcf_hist.append(ocf[i] - capex[i])
        fcf_consistency = safe_div(len([x for x in fcf_hist if x > 0]), len(fcf_hist)) if fcf_hist else None

        return {
            "revenue_cagr": revenue_cagr,
            "ebitda_cagr": ebitda_cagr,
            "debt_to_ebitda_change": dte_change,
            "interest_coverage_change": ic_change,
            "fcf_consistency": fcf_consistency,
        }

    @staticmethod
    def score_debt_to_ebitda(x):
        if x is None: return None
        return 20 if x <= 1 else 17 if x <= 2 else 14 if x <= 3 else 10 if x <= 4 else 6 if x <= 5 else 2

    @staticmethod
    def score_interest_coverage(x):
        if x is None: return None
        return 18 if x >= 8 else 16 if x >= 5 else 12 if x >= 3 else 7 if x >= 1.5 else 3 if x >= 1 else 0

    @staticmethod
    def score_fcf_to_debt(x):
        if x is None: return None
        return 15 if x >= .25 else 12 if x >= .15 else 8 if x >= .08 else 5 if x >= .02 else 3 if x >= 0 else 0

    @staticmethod
    def score_current_ratio(x):
        if x is None: return None
        return 8 if x >= 2 else 7 if x >= 1.5 else 5 if x >= 1.2 else 3 if x >= 1 else 1 if x >= .8 else 0

    @staticmethod
    def score_quick_ratio(x):
        if x is None: return None
        return 6 if x >= 1.5 else 5 if x >= 1.2 else 4 if x >= 1 else 2 if x >= .8 else 0

    @staticmethod
    def score_ebitda_margin(x):
        if x is None: return None
        return 11 if x >= .25 else 9 if x >= .15 else 6 if x >= .08 else 3 if x >= .03 else 0

    @staticmethod
    def score_debt_to_capital(x):
        if x is None: return None
        return 10 if x <= .25 else 8 if x <= .40 else 6 if x <= .55 else 3 if x <= .70 else 0

    @staticmethod
    def score_cash_to_debt(x):
        if x is None: return None
        return 6 if x >= .50 else 5 if x >= .25 else 3 if x >= .10 else 1 if x >= .05 else 0

    @staticmethod
    def score_asset_coverage(x):
        if x is None: return None
        return 6 if x >= 3 else 5 if x >= 2 else 3 if x >= 1.5 else 1 if x >= 1 else 0

    def score_components(self, ratios: Dict[str, Optional[float]]) -> Tuple[Dict[str, Optional[float]], float, float, float]:
        funcs = {
            "debt_to_ebitda": self.score_debt_to_ebitda,
            "interest_coverage": self.score_interest_coverage,
            "fcf_to_debt": self.score_fcf_to_debt,
            "current_ratio": self.score_current_ratio,
            "quick_ratio": self.score_quick_ratio,
            "ebitda_margin": self.score_ebitda_margin,
            "debt_to_capital": self.score_debt_to_capital,
            "cash_to_debt": self.score_cash_to_debt,
            "asset_coverage": self.score_asset_coverage,
        }
        scores, earned, available = {}, 0.0, 0.0
        total_max = sum(self.MAX_POINTS.values())
        for metric, func in funcs.items():
            s = func(ratios.get(metric))
            scores[f"{metric}_score"] = s
            if s is not None:
                earned += s
                available += self.MAX_POINTS[metric]
        base = (earned / available) * 100 if available else 0.0
        coverage = available / total_max if total_max else 0.0
        return scores, base, coverage, available

    @staticmethod
    def assign_rating(score: float) -> str:
        return "A" if score >= 90 else "BBB" if score >= 80 else "BB" if score >= 70 else "B" if score >= 60 else "CCC" if score >= 50 else "Distressed"

    @staticmethod
    def estimate_pd(rating: str) -> float:
        return {"A": .005, "BBB": .015, "BB": .040, "B": .090, "CCC": .200, "Distressed": .400}.get(rating, .400)

    @staticmethod
    def estimate_lgd(asset_coverage: Optional[float]) -> float:
        if asset_coverage is None: return .60
        return .25 if asset_coverage >= 3 else .35 if asset_coverage >= 2 else .45 if asset_coverage >= 1.5 else .60 if asset_coverage >= 1 else .75

    @staticmethod
    def qualitative_adjustment(b: Borrower) -> float:
        return ((3 - b.sector_risk_score) * 1.5) + ((b.management_score - 3) * 1.5) + ((3 - b.customer_concentration_score) * 1.0)

    @staticmethod
    def trend_adjustment(t: Dict[str, Optional[float]]) -> float:
        adj = 0.0
        if t.get("revenue_cagr") is not None:
            adj += 1.5 if t["revenue_cagr"] > .05 else -1.5 if t["revenue_cagr"] < -.05 else 0
        if t.get("ebitda_cagr") is not None:
            adj += 2.0 if t["ebitda_cagr"] > .05 else -2.5 if t["ebitda_cagr"] < -.05 else 0
        if t.get("debt_to_ebitda_change") is not None:
            adj += 2.0 if t["debt_to_ebitda_change"] < -.5 else -2.0 if t["debt_to_ebitda_change"] > .5 else 0
        if t.get("interest_coverage_change") is not None:
            adj += 1.5 if t["interest_coverage_change"] > 1 else -1.5 if t["interest_coverage_change"] < -1 else 0
        if t.get("fcf_consistency") is not None:
            adj += 2.0 if t["fcf_consistency"] >= 1 else -3.0 if t["fcf_consistency"] < .5 else 0
        return adj

    @staticmethod
    def coverage_penalty(data_coverage: float) -> float:
        return 0.0 if data_coverage >= .80 else min(8.0, (.80 - data_coverage) * 20)

    def flags(self, b: Borrower, r: Dict[str, Optional[float]], t: Dict[str, Optional[float]], data_coverage: float) -> List[str]:
        out = []
        if data_coverage < .70: out.append("Low financial data coverage")
        if b.ebitda is None: out.append("Missing EBITDA")
        elif b.ebitda <= 0: out.append("Negative or zero EBITDA")
        if r.get("debt_to_ebitda") is not None and r["debt_to_ebitda"] > 4: out.append("High leverage")
        if r.get("net_debt_to_ebitda") is not None and r["net_debt_to_ebitda"] > 4: out.append("High net leverage")
        if r.get("interest_coverage") is not None and r["interest_coverage"] < 1.5: out.append("Weak interest coverage")
        if r.get("free_cash_flow") is not None and r["free_cash_flow"] < 0: out.append("Negative free cash flow")
        if r.get("current_ratio") is not None and r["current_ratio"] < 1: out.append("Liquidity pressure")
        if r.get("quick_ratio") is not None and r["quick_ratio"] < .8: out.append("Weak quick ratio")
        if r.get("cash_to_debt") is not None and r["cash_to_debt"] < .05: out.append("Low cash cushion")
        if r.get("asset_coverage") is not None and r["asset_coverage"] < 1: out.append("Weak asset coverage")
        if b.total_equity is not None and b.total_equity < 0: out.append("Negative book equity")
        if t.get("revenue_cagr") is not None and t["revenue_cagr"] < -.05: out.append("Revenue contraction trend")
        if t.get("ebitda_cagr") is not None and t["ebitda_cagr"] < -.05: out.append("EBITDA contraction trend")
        if t.get("fcf_consistency") is not None and t["fcf_consistency"] < .5: out.append("Inconsistent free cash flow generation")
        return out

    @staticmethod
    def strengths(r: Dict[str, Optional[float]], t: Dict[str, Optional[float]]) -> List[str]:
        out = []
        if r.get("debt_to_ebitda") is not None and r["debt_to_ebitda"] <= 2: out.append("Low leverage")
        if r.get("net_debt_to_ebitda") is not None and r["net_debt_to_ebitda"] <= 1.5: out.append("Low net leverage")
        if r.get("interest_coverage") is not None and r["interest_coverage"] >= 5: out.append("Strong interest coverage")
        if r.get("fcf_to_debt") is not None and r["fcf_to_debt"] >= .15: out.append("Strong FCF-to-debt")
        if r.get("current_ratio") is not None and r["current_ratio"] >= 1.5: out.append("Good liquidity")
        if r.get("ebitda_margin") is not None and r["ebitda_margin"] >= .15: out.append("Healthy EBITDA margin")
        if r.get("cash_to_debt") is not None and r["cash_to_debt"] >= .25: out.append("Strong cash cushion")
        if r.get("asset_coverage") is not None and r["asset_coverage"] >= 2: out.append("Strong asset coverage")
        if t.get("fcf_consistency") is not None and t["fcf_consistency"] >= 1: out.append("Consistent positive free cash flow")
        return out

    @staticmethod
    def rule_feedback(b: Borrower, rating: str, score: float, flags: List[str], strengths: List[str]) -> str:
        view = "screens as a stronger credit profile" if rating in ["A", "BBB"] else "screens as a moderate credit risk" if rating in ["BB", "B"] else "screens as an elevated credit risk"
        st = ", ".join(strengths[:3]) if strengths else "limited clearly identifiable strengths"
        fl = ", ".join(flags[:3]) if flags else "no major model-generated red flags"
        return f"{b.ticker} / {b.company_name} {view} with an internal {rating} rating and score of {score:.1f}. Key positives include {st}. Main watch items include {fl}."

    def evaluate(self, b: Borrower, include_stress: bool = True, apply_trend: bool = True) -> Dict[str, Any]:
        ratios = self.calculate_ratios(b)
        trends = self.calculate_trends(b)
        score_components, base_score, data_coverage, available_max = self.score_components(ratios)
        q_adj = self.qualitative_adjustment(b)
        t_adj = self.trend_adjustment(trends) if apply_trend else 0.0
        penalty = self.coverage_penalty(data_coverage)
        final_score = clamp(base_score + q_adj + t_adj - penalty)
        rating = self.assign_rating(final_score)
        pd_est = self.estimate_pd(rating)
        lgd = self.estimate_lgd(ratios.get("asset_coverage"))
        exposure = b.total_debt if b.total_debt is not None else 0.0
        expected_loss = exposure * pd_est * lgd
        flags = self.flags(b, ratios, trends, data_coverage)
        strengths = self.strengths(ratios, trends)
        result = {
            "ticker": b.ticker,
            "company_name": b.company_name,
            "sector": b.sector,
            "industry": b.industry,
            "base_score": round(base_score, 2),
            "qualitative_adjustment": round(q_adj, 2),
            "trend_adjustment": round(t_adj, 2),
            "coverage_penalty": round(penalty, 2),
            "data_coverage": round(data_coverage, 4),
            "available_max_points": round(available_max, 2),
            "final_score": round(final_score, 2),
            "internal_rating": rating,
            "probability_of_default": pd_est,
            "loss_given_default": lgd,
            "expected_loss": expected_loss,
            "ratios": ratios,
            "trends": trends,
            "score_components": score_components,
            "flags": flags,
            "strengths": strengths,
            "rule_based_feedback": self.rule_feedback(b, rating, final_score, flags, strengths),
        }
        if include_stress:
            result["stress_tests"] = self.run_stress_tests(b, result)
        return result

    def stressed_borrower(self, b: Borrower, case_name: str, revenue_shock: float, ebitda_shock: float, ebit_shock: float, interest_shock: float, ocf_shock: float) -> Borrower:
        def shock(x, s): return x * (1 + s) if x is not None else None
        return Borrower(
            ticker=b.ticker,
            company_name=f"{b.company_name} - {case_name}",
            sector=b.sector,
            industry=b.industry,
            revenue=shock(b.revenue, revenue_shock),
            ebitda=shock(b.ebitda, ebitda_shock),
            ebit=shock(b.ebit, ebit_shock),
            interest_expense=shock(b.interest_expense, interest_shock),
            total_debt=b.total_debt,
            cash=b.cash,
            total_assets=b.total_assets,
            total_equity=b.total_equity,
            current_assets=b.current_assets,
            current_liabilities=b.current_liabilities,
            inventory=b.inventory,
            operating_cash_flow=shock(b.operating_cash_flow, ocf_shock),
            capex=b.capex,
            history=b.history,
            sector_risk_score=b.sector_risk_score,
            management_score=b.management_score,
            customer_concentration_score=b.customer_concentration_score,
        )

    def run_stress_tests(self, b: Borrower, base_result: Dict[str, Any]) -> List[Dict[str, Any]]:
        cases = [
            ("Moderate Downside", -.05, -.12, -.15, .15, -.10),
            ("Severe Downside", -.12, -.25, -.30, .30, -.25),
        ]
        out = []
        for case, rev, ebitda, ebit, interest, ocf in cases:
            sb = self.stressed_borrower(b, case, rev, ebitda, ebit, interest, ocf)
            sr = self.evaluate(sb, include_stress=False, apply_trend=False)
            out.append({
                "ticker": b.ticker,
                "company_name": b.company_name,
                "stress_case": case,
                "base_score": base_result["final_score"],
                "stress_score": sr["final_score"],
                "score_change": round(sr["final_score"] - base_result["final_score"], 2),
                "base_rating": base_result["internal_rating"],
                "stress_rating": sr["internal_rating"],
                "stress_pd": sr["probability_of_default"],
                "stress_expected_loss": sr["expected_loss"],
                "stress_debt_to_ebitda": sr["ratios"]["debt_to_ebitda"],
                "stress_interest_coverage": sr["ratios"]["interest_coverage"],
                "stress_fcf_to_debt": sr["ratios"]["fcf_to_debt"],
            })
        return out


def flatten_result(r: Dict[str, Any]) -> Dict[str, Any]:
    ratios, trends = r["ratios"], r["trends"]
    return {
        "ticker": r["ticker"],
        "company_name": r["company_name"],
        "sector": r["sector"],
        "industry": r["industry"],
        "base_score": r["base_score"],
        "qualitative_adjustment": r["qualitative_adjustment"],
        "trend_adjustment": r["trend_adjustment"],
        "coverage_penalty": r["coverage_penalty"],
        "data_coverage": r["data_coverage"],
        "available_max_points": r["available_max_points"],
        "final_score": r["final_score"],
        "internal_rating": r["internal_rating"],
        "probability_of_default": r["probability_of_default"],
        "loss_given_default": r["loss_given_default"],
        "expected_loss": r["expected_loss"],
        **{k: ratios.get(k) for k in [
            "debt_to_ebitda", "net_debt_to_ebitda", "interest_coverage", "ebitda_interest_coverage",
            "fcf_to_debt", "current_ratio", "quick_ratio", "ebitda_margin", "debt_to_capital",
            "cash_to_debt", "asset_coverage", "free_cash_flow", "net_debt"
        ]},
        **{k: trends.get(k) for k in [
            "revenue_cagr", "ebitda_cagr", "debt_to_ebitda_change", "interest_coverage_change", "fcf_consistency"
        ]},
        "strengths": "; ".join(r["strengths"]),
        "flags": "; ".join(r["flags"]),
        "rule_based_feedback": r["rule_based_feedback"],
    }


def flatten_borrower(b: Borrower) -> Dict[str, Any]:
    d = asdict(b)
    d["history"] = json.dumps(d.get("history", {}))
    return d


def rule_based_portfolio_feedback(df: pd.DataFrame) -> str:
    if df.empty:
        return "No credit results were generated."
    lines = [
        "CORPORATE CREDIT PORTFOLIO REVIEW",
        "=" * 80,
        f"Generated: {now_iso()} UTC",
        "",
        f"Number of evaluated firms: {len(df)}",
        f"Average score: {df['final_score'].mean():.2f}",
        f"Median score: {df['final_score'].median():.2f}",
        "",
        "Rating distribution:",
    ]
    for rating, count in df["internal_rating"].value_counts().to_dict().items():
        lines.append(f"- {rating}: {count}")
    lines += ["", "Top 5 strongest credits:"]
    for _, row in df.sort_values("final_score", ascending=False).head(5).iterrows():
        lines.append(f"- {row['ticker']}: {row['internal_rating']} / {row['final_score']:.1f} | {row['company_name']}")
    lines += ["", "Top 5 weakest credits:"]
    for _, row in df.sort_values("final_score", ascending=True).head(5).iterrows():
        flags = row["flags"] if isinstance(row["flags"], str) and row["flags"] else "No major flags"
        lines.append(f"- {row['ticker']}: {row['internal_rating']} / {row['final_score']:.1f} | Flags: {flags}")
    lines += ["", "Watchlist names:"]
    watchlist = df[(df["internal_rating"].isin(["CCC", "Distressed"])) | (df["flags"].astype(str).str.len() > 0)].sort_values("final_score")
    if watchlist.empty:
        lines.append("- No major watchlist names identified.")
    else:
        for _, row in watchlist.head(10).iterrows():
            lines.append(f"- {row['ticker']}: {row['internal_rating']} / {row['final_score']:.1f} | {row['flags']}")
    lines += ["", "Firm-level one-sentence feedback:"]
    for _, row in df.sort_values("final_score", ascending=False).iterrows():
        lines.append(f"- {row['rule_based_feedback']}")
    return "\n".join(lines)


def prepare_ai_payload(df: pd.DataFrame, max_firms: int = 20) -> List[Dict[str, Any]]:
    cols = [
        "ticker", "company_name", "sector", "final_score", "internal_rating", "probability_of_default",
        "loss_given_default", "debt_to_ebitda", "net_debt_to_ebitda", "interest_coverage",
        "fcf_to_debt", "current_ratio", "quick_ratio", "ebitda_margin", "debt_to_capital",
        "cash_to_debt", "asset_coverage", "revenue_cagr", "ebitda_cagr", "fcf_consistency",
        "strengths", "flags"
    ]
    cols = [c for c in cols if c in df.columns]
    compact = df[cols].sort_values("final_score", ascending=False).head(max_firms)
    return compact.replace({np.nan: None}).to_dict(orient="records")


def generate_ai_credit_feedback(df: pd.DataFrame) -> str:
    api_key = os.getenv("OPENAI_API_KEY")
    if not api_key:
        return "AI feedback skipped: OPENAI_API_KEY was not found."
    try:
        from openai import OpenAI
    except ImportError:
        return "AI feedback skipped: openai package is not installed. Run: pip install openai"

    model_name = os.getenv("OPENAI_MODEL", "gpt-5.4-mini")
    try:
        max_tokens = min(int(os.getenv("OPENAI_MAX_OUTPUT_TOKENS", "900")), 1200)
    except ValueError:
        max_tokens = 900

    payload = prepare_ai_payload(df)
    prompt = f"""
You are a corporate credit analyst.

Use only the structured model output below. Do not use outside news, web search, market prices, or information not included in the data.

Write a concise professional credit memo.

Rules:
- Do not invent missing data.
- Do not recalculate ratios.
- Do not claim this is an official agency rating.
- Keep the memo under 750 words.
- Focus on leverage, liquidity, coverage, cash flow, trend quality, and watchlist risk.
- Mention that the ratings are internal model scores.
- Include: portfolio summary, strongest credits, weakest credits, watchlist themes, and one-sentence view for each firm.

Structured model output:
{json.dumps(payload, indent=2)}
"""
    try:
        client = OpenAI(api_key=api_key)
        response = client.responses.create(model=model_name, input=prompt, max_output_tokens=max_tokens)
        return getattr(response, "output_text", None) or str(response)
    except Exception as exc:
        return f"AI feedback failed: {exc}"


def save_sqlite(output_dir: Path, borrowers: pd.DataFrame, credit: pd.DataFrame, stress: pd.DataFrame) -> Path:
    db_path = output_dir / "corporate_credit_model.db"
    with sqlite3.connect(db_path) as conn:
        borrowers.to_sql("raw_borrowers", conn, if_exists="replace", index=False)
        credit.to_sql("credit_scores", conn, if_exists="replace", index=False)
        stress.to_sql("stress_tests", conn, if_exists="replace", index=False)
    return db_path


def evaluate_tickers(tickers: List[str], output_dir: Path, cache_dir: Path, cache_days: int, refresh: bool, sleep_seconds: float, use_ai: bool) -> pd.DataFrame:
    ensure_dir(output_dir)
    ensure_dir(cache_dir)
    model = CorporateCreditModel()
    borrower_rows, credit_rows, stress_rows = [], [], []

    for ticker in tickers:
        borrower = pull_or_load_borrower(ticker, cache_dir, cache_days, refresh, sleep_seconds)
        if borrower is None:
            continue
        borrower_rows.append(flatten_borrower(borrower))
        result = model.evaluate(borrower)
        credit_rows.append(flatten_result(result))
        stress_rows.extend(result.get("stress_tests", []))

    borrower_df = pd.DataFrame(borrower_rows)
    credit_df = pd.DataFrame(credit_rows)
    stress_df = pd.DataFrame(stress_rows)

    if credit_df.empty:
        print("No credit results generated.")
        return credit_df

    credit_df = credit_df.sort_values("final_score", ascending=False).reset_index(drop=True)

    credit_path = output_dir / "corporate_credit_results.csv"
    stress_path = output_dir / "stress_test_results.csv"
    raw_path = output_dir / "raw_borrowers.csv"
    rule_path = output_dir / "rule_based_credit_feedback.txt"
    ai_path = output_dir / "ai_credit_feedback.txt"

    credit_df.to_csv(credit_path, index=False)
    stress_df.to_csv(stress_path, index=False)
    borrower_df.to_csv(raw_path, index=False)

    with open(rule_path, "w", encoding="utf-8") as f:
        f.write(rule_based_portfolio_feedback(credit_df))

    ai_text = generate_ai_credit_feedback(credit_df) if use_ai else "AI feedback not run. Run with: python corporate_credit_model_full.py --ai\n"
    with open(ai_path, "w", encoding="utf-8") as f:
        f.write(ai_text)

    db_path = save_sqlite(output_dir, borrower_df, credit_df, stress_df)

    print("\n" + "=" * 90)
    print("Corporate Credit Evaluation Complete")
    print("=" * 90)
    print(f"Credit results CSV:  {credit_path}")
    print(f"Stress test CSV:     {stress_path}")
    print(f"Raw borrower CSV:    {raw_path}")
    print(f"Rule feedback TXT:   {rule_path}")
    print(f"AI feedback TXT:     {ai_path}")
    print(f"SQLite database:     {db_path}")
    print("=" * 90 + "\n")

    display_cols = ["ticker", "company_name", "sector", "final_score", "internal_rating", "probability_of_default", "debt_to_ebitda", "interest_coverage", "fcf_to_debt", "current_ratio", "flags"]
    display_cols = [c for c in display_cols if c in credit_df.columns]
    print(credit_df[display_cols].to_string(index=False))
    return credit_df


def load_tickers_file(path: Path) -> List[str]:
    if not path.exists():
        raise FileNotFoundError(f"Ticker file not found: {path}")
    with open(path, "r", encoding="utf-8") as f:
        return normalize_tickers([line.strip() for line in f if line.strip()])


def parse_args() -> argparse.Namespace:
    p = argparse.ArgumentParser(description="Corporate Credit Evaluation Model: 20 firms, trends, stress tests, AI memo, SQLite.")
    p.add_argument("--tickers", nargs="*", default=None, help="Optional custom tickers, e.g. --tickers AAPL MSFT AMZN")
    p.add_argument("--tickers-file", default=None, help="Text file with one ticker per line.")
    p.add_argument("--output-dir", default="outputs", help="Output directory.")
    p.add_argument("--cache-dir", default="cache/yfinance", help="Cache directory.")
    p.add_argument("--cache-days", type=int, default=7, help="Days before cache refresh.")
    p.add_argument("--refresh", action="store_true", help="Force refresh of yfinance data.")
    p.add_argument("--sleep", type=float, default=0.75, help="Seconds between yfinance pulls.")
    p.add_argument("--ai", action="store_true", help="Generate one capped AI portfolio memo. Requires OPENAI_API_KEY.")
    return p.parse_args()


def main() -> None:
    args = parse_args()
    if args.tickers_file:
        tickers = load_tickers_file(Path(args.tickers_file))
    elif args.tickers:
        tickers = normalize_tickers(args.tickers)
    else:
        tickers = DEFAULT_TICKERS

    if not tickers:
        print("No tickers provided.")
        sys.exit(1)

    evaluate_tickers(
        tickers=tickers,
        output_dir=Path(args.output_dir),
        cache_dir=Path(args.cache_dir),
        cache_days=args.cache_days,
        refresh=args.refresh,
        sleep_seconds=args.sleep,
        use_ai=args.ai,
    )


if __name__ == "__main__":
    main()
