# API Endpoint Mapping: Financial Datasets vs Massive.com

## Purpose

This document provides a side-by-side comparison of Financial Datasets API endpoints and placeholder mappings for Massive.com API endpoints (to be updated once documentation is available).

## Status: ⚠️ INCOMPLETE

**Massive.com API documentation is currently unavailable.** All Massive.com endpoint information below are educated guesses based on common financial API patterns and need to be verified against actual API documentation.

---

## 1. Stock Prices (OHLCV Data)

### Financial Datasets API

**Endpoint:** `GET /prices/`

**URL Example:**
```
https://api.financialdatasets.ai/prices/?ticker=AAPL&interval=day&interval_multiplier=1&start_date=2024-01-01&end_date=2024-12-31
```

**Parameters:**
- `ticker` (required): Stock symbol
- `interval` (required): Time interval (day, week, month)
- `interval_multiplier` (required): Multiplier for interval
- `start_date` (required): ISO date format
- `end_date` (required): ISO date format

**Response:**
```json
{
  "ticker": "AAPL",
  "prices": [
    {
      "open": 179.55,
      "close": 179.66,
      "high": 180.53,
      "low": 177.38,
      "volume": 73450582,
      "time": "2024-03-01T05:00:00Z"
    }
  ]
}
```

### Massive.com API (Placeholder)

**Estimated Endpoint:** `GET /quotes/historical` *(unverified)*

**Possible URL:**
```
https://api.massive.com/quotes/historical?symbol=AAPL&from=2024-01-01&to=2024-12-31&interval=1d
```

**Estimated Parameters:**
- TBD - awaiting documentation

**Estimated Response:**
- TBD - awaiting documentation

**Data Transformation Required:**
- Map field names (open, close, high, low, volume, timestamp)
- Convert date/time format if different
- Handle timezone differences

---

## 2. Financial Metrics

### Financial Datasets API

**Endpoint:** `GET /financial-metrics/`

**URL Example:**
```
https://api.financialdatasets.ai/financial-metrics/?ticker=AAPL&report_period_lte=2024-12-31&limit=10&period=ttm
```

**Parameters:**
- `ticker` (required): Stock symbol
- `report_period_lte` (required): Report period less than or equal to date
- `limit` (optional): Number of results (default: 10)
- `period` (optional): ttm, annual, quarterly

**Response Fields (40+ metrics):**
- Market cap, Enterprise value
- P/E ratio, P/B ratio, P/S ratio
- EV/EBITDA, EV/Revenue
- Profit margins (gross, operating, net)
- Returns (ROE, ROA, ROIC)
- Liquidity ratios (current, quick, cash)
- Leverage ratios (debt/equity, debt/assets)
- Growth rates (revenue, earnings, FCF)
- Per-share metrics (EPS, book value, FCF)

### Massive.com API (Placeholder)

**Estimated Endpoint:** TBD

**Requirements:**
- Must provide all 40+ financial metrics currently used
- Must support TTM, annual, and quarterly periods
- Must provide historical data (not just current)

**Alternative Approach:**
- May require multiple endpoint calls
- May need calculation of derived metrics

**Critical Question:**
- Does Massive.com provide comprehensive financial metrics, or just raw financial statement data?

---

## 3. Financial Statement Line Items

### Financial Datasets API

**Endpoint:** `POST /financials/search/line-items`

**Request Body:**
```json
{
  "tickers": ["AAPL"],
  "line_items": [
    "revenue",
    "cost_of_revenue",
    "operating_expenses",
    "net_income"
  ],
  "end_date": "2024-12-31",
  "period": "ttm",
  "limit": 10
}
```

**Response:**
```json
{
  "search_results": [
    {
      "ticker": "AAPL",
      "report_period": "2024-Q4",
      "period": "quarterly",
      "currency": "USD",
      "revenue": 123456789000,
      "cost_of_revenue": 75000000000,
      "operating_expenses": 25000000000,
      "net_income": 23456789000
    }
  ]
}
```

### Massive.com API (Placeholder)

**Estimated Endpoint:** TBD

**Critical Requirements:**
- Flexible search by line item name
- Support for custom/calculated line items
- Historical data across multiple periods

**Potential Issues:**
- Different accounting terminology
- Limited customization
- May require parsing full statements

---

## 4. Insider Trades

### Financial Datasets API

**Endpoint:** `GET /insider-trades/`

**URL Example:**
```
https://api.financialdatasets.ai/insider-trades/?ticker=AAPL&filing_date_lte=2024-12-31&filing_date_gte=2024-01-01&limit=1000
```

**Parameters:**
- `ticker` (required): Stock symbol
- `filing_date_lte` (optional): Filing date less than or equal to
- `filing_date_gte` (optional): Filing date greater than or equal to
- `limit` (optional): Max results per page

**Response Fields:**
- Insider name, title
- Transaction date, filing date
- Transaction type (buy/sell)
- Shares traded, price per share
- Total transaction value
- Shares owned before/after

### Massive.com API (Placeholder)

**Estimated Endpoint:** TBD

**Critical Requirements:**
- SEC Form 4 data
- Director and officer transactions
- Historical data
- Pagination support

**Data Quality Concerns:**
- Timeliness of data
- Accuracy of calculations
- Coverage (all insiders or just executives?)

---

## 5. Company News

### Financial Datasets API

**Endpoint:** `GET /news/`

**URL Example:**
```
https://api.financialdatasets.ai/news/?ticker=AAPL&start_date=2024-01-01&end_date=2024-12-31&limit=1000
```

**Parameters:**
- `ticker` (required): Stock symbol
- `start_date` (optional): News date start
- `end_date` (required): News date end
- `limit` (optional): Max results

**Response Fields:**
- Title, author, source
- Publication date
- Article URL
- Sentiment (positive/negative/neutral)

### Massive.com API (Placeholder)

**Estimated Endpoint:** TBD

**Critical Requirements:**
- News aggregation from multiple sources
- Sentiment analysis
- Recency (real-time or delayed?)
- Historical archive

**Potential Issues:**
- News sources may differ
- Sentiment scoring methodology
- Duplication handling
- Paywall/access restrictions

---

## 6. Company Facts

### Financial Datasets API

**Endpoint:** `GET /company/facts/`

**URL Example:**
```
https://api.financialdatasets.ai/company/facts/?ticker=AAPL
```

**Response Fields:**
- Company name
- Ticker symbol
- CIK number
- Industry, sector, category
- Exchange
- Market cap
- Location, website
- Number of employees
- SIC code and industry

### Massive.com API (Placeholder)

**Estimated Endpoint:** TBD

**Critical Requirements:**
- Company profile data
- Current market cap
- Sector/industry classification
- Up-to-date information

---

## Data Validation Checklist

When Massive.com API documentation becomes available, validate:

### Coverage
- [ ] All 6 endpoint types available
- [ ] Same or better data coverage
- [ ] Historical data depth matches needs
- [ ] Update frequency acceptable

### Data Quality
- [ ] Price data matches other sources
- [ ] Financial metrics calculations verified
- [ ] News sentiment scores reasonable
- [ ] Insider trade data complete

### Performance
- [ ] Response times acceptable (< 2s)
- [ ] Rate limits sufficient for use case
- [ ] Pagination efficient
- [ ] Bulk requests supported

### Cost
- [ ] Pricing model understood
- [ ] Cost per request calculated
- [ ] Volume discounts available
- [ ] Overage charges known

### Features
- [ ] Authentication method documented
- [ ] Error handling clear
- [ ] API versioning strategy
- [ ] Deprecation policy
- [ ] SLA/uptime guarantees

---

## Implementation Notes

### Current Usage Patterns

1. **Price Data:**
   - Fetched daily for backtesting
   - Used by technical analysis agents
   - Cached aggressively

2. **Financial Metrics:**
   - Fetched quarterly/annually
   - Used by value investing agents
   - Less frequent updates

3. **News:**
   - Fetched for recent date ranges
   - Used by sentiment agents
   - High volume during analysis

4. **Insider Trades:**
   - Fetched for specific date ranges
   - Used by specific agents (Michael Burry, etc.)
   - Moderate volume

### Caching Strategy

Current cache keys use format: `{ticker}_{param1}_{param2}_{...}`

Example: `AAPL_2024-01-01_2024-12-31`

Cache invalidation:
- Price data: Daily
- Financial metrics: Quarterly
- News: Daily
- Insider trades: Weekly

### Rate Limiting

Current implementation:
- 3 retries with linear backoff
- Wait times: 60s, 90s, 120s
- Handles 429 status codes

Must verify Massive.com rate limits align with usage patterns.

---

## Questions for Massive.com Team

When contacting Massive.com, ask:

1. **Documentation:**
   - Where is the API documentation?
   - Is there an API reference/OpenAPI spec?
   - Are there code examples in Python?

2. **Data Coverage:**
   - Which data types are available? (See 6 endpoints above)
   - Historical data depth for each endpoint?
   - Update frequency and latency?

3. **Authentication:**
   - How to obtain API keys?
   - Authentication method (header, query param, OAuth)?
   - Key rotation policy?

4. **Pricing:**
   - Pricing model (per request, subscription, data volume)?
   - Free tier or trial available?
   - Enterprise pricing for high volume?

5. **Technical:**
   - Rate limits per endpoint?
   - Pagination approach?
   - Webhook support for real-time data?
   - Bulk data export capabilities?

6. **Reliability:**
   - SLA/uptime guarantees?
   - Status page?
   - Support channels?
   - Incident history?

7. **Migration:**
   - Migration support available?
   - Sandbox/test environment?
   - Data validation tools?

---

## Appendix: Data Model Comparison

### Current Data Models (Pydantic)

All our data models are in `src/data/models.py`:

```python
class Price(BaseModel):
    open: float
    close: float
    high: float
    low: float
    volume: int
    time: str

class FinancialMetrics(BaseModel):
    ticker: str
    report_period: str
    period: str
    currency: str
    market_cap: float | None
    # ... 40+ more fields

class CompanyNews(BaseModel):
    ticker: str
    title: str
    author: str
    source: str
    date: str
    url: str
    sentiment: str | None

class InsiderTrade(BaseModel):
    ticker: str
    name: str | None
    title: str | None
    transaction_date: str | None
    transaction_shares: float | None
    transaction_price_per_share: float | None
    # ... more fields
```

These models must be preserved or adapted to maintain compatibility with existing agent code.

---

## Conclusion

This endpoint mapping document will be updated once Massive.com API documentation becomes available. All information about Massive.com endpoints is currently speculative and must be verified.

**Next Steps:**
1. Obtain Massive.com API documentation
2. Update this document with actual endpoints
3. Create detailed transformation logic
4. Validate data quality with sample requests
5. Proceed with migration planning

**Last Updated:** December 14, 2024
**Status:** Awaiting Massive.com API Documentation
