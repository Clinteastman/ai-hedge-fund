# Migration Strategy: Financial Datasets to Massive.com API

## Overview

This document provides a technical migration strategy for swapping from Financial Datasets API to Massive.com API, assuming the Massive.com API becomes available and accessible.

## Strategy: Provider Abstraction Pattern

To enable flexible switching between data providers and minimize future migration pain, we recommend implementing an abstraction layer.

### Benefits

1. **Easy Provider Switching:** Change providers with configuration
2. **Multiple Provider Support:** Use different providers for different data types
3. **Gradual Migration:** Test new providers without touching agent code
4. **Fallback Support:** Automatic failover to backup provider
5. **Testing:** Easy to mock and test providers independently

## Phase 1: Create Provider Abstraction (Week 1-2)

### Step 1.1: Define Abstract Interface

Create `src/data/providers/base.py`:

```python
from abc import ABC, abstractmethod
from typing import Optional
from src.data.models import (
    Price,
    FinancialMetrics,
    LineItem,
    InsiderTrade,
    CompanyNews,
)


class FinancialDataProvider(ABC):
    """Abstract base class for financial data providers."""
    
    def __init__(self, api_key: Optional[str] = None):
        self.api_key = api_key
    
    @abstractmethod
    def get_prices(
        self, 
        ticker: str, 
        start_date: str, 
        end_date: str
    ) -> list[Price]:
        """Fetch historical price data."""
        pass
    
    @abstractmethod
    def get_financial_metrics(
        self,
        ticker: str,
        end_date: str,
        period: str = "ttm",
        limit: int = 10,
    ) -> list[FinancialMetrics]:
        """Fetch financial metrics and ratios."""
        pass
    
    @abstractmethod
    def search_line_items(
        self,
        ticker: str,
        line_items: list[str],
        end_date: str,
        period: str = "ttm",
        limit: int = 10,
    ) -> list[LineItem]:
        """Search for specific financial statement line items."""
        pass
    
    @abstractmethod
    def get_insider_trades(
        self,
        ticker: str,
        end_date: str,
        start_date: Optional[str] = None,
        limit: int = 1000,
    ) -> list[InsiderTrade]:
        """Fetch insider trading data."""
        pass
    
    @abstractmethod
    def get_company_news(
        self,
        ticker: str,
        end_date: str,
        start_date: Optional[str] = None,
        limit: int = 1000,
    ) -> list[CompanyNews]:
        """Fetch company news articles."""
        pass
    
    @abstractmethod
    def get_market_cap(
        self,
        ticker: str,
        end_date: str,
    ) -> Optional[float]:
        """Fetch current or historical market cap."""
        pass
```

### Step 1.2: Wrap Current Implementation

Create `src/data/providers/financial_datasets.py`:

```python
import os
import logging
import requests
import time
from typing import Optional
from src.data.providers.base import FinancialDataProvider
from src.data.models import (
    Price, PriceResponse,
    FinancialMetrics, FinancialMetricsResponse,
    LineItem, LineItemResponse,
    InsiderTrade, InsiderTradeResponse,
    CompanyNews, CompanyNewsResponse,
    CompanyFactsResponse,
)

logger = logging.getLogger(__name__)


class FinancialDatasetsProvider(FinancialDataProvider):
    """Financial Datasets AI provider implementation."""
    
    BASE_URL = "https://api.financialdatasets.ai"
    
    def __init__(self, api_key: Optional[str] = None):
        super().__init__(api_key or os.environ.get("FINANCIAL_DATASETS_API_KEY"))
    
    def _get_headers(self) -> dict:
        """Get headers for API requests."""
        headers = {}
        if self.api_key:
            headers["X-API-KEY"] = self.api_key
        return headers
    
    def _make_request(
        self, 
        url: str, 
        method: str = "GET", 
        json_data: dict = None,
        max_retries: int = 3
    ) -> requests.Response:
        """Make API request with retry logic."""
        headers = self._get_headers()
        
        for attempt in range(max_retries + 1):
            if method.upper() == "POST":
                response = requests.post(url, headers=headers, json=json_data)
            else:
                response = requests.get(url, headers=headers)
            
            if response.status_code == 429 and attempt < max_retries:
                delay = 60 + (30 * attempt)
                logger.warning(f"Rate limited. Waiting {delay}s before retry...")
                time.sleep(delay)
                continue
            
            return response
    
    def get_prices(self, ticker: str, start_date: str, end_date: str) -> list[Price]:
        """Fetch price data."""
        url = f"{self.BASE_URL}/prices/?ticker={ticker}&interval=day&interval_multiplier=1&start_date={start_date}&end_date={end_date}"
        response = self._make_request(url)
        
        if response.status_code != 200:
            return []
        
        try:
            price_response = PriceResponse(**response.json())
            return price_response.prices
        except Exception as e:
            logger.error(f"Error parsing price response for {ticker}: {e}")
            return []
    
    # ... implement other methods similarly
    # (Copy from existing src/tools/api.py)
```

### Step 1.3: Create Massive.com Provider Stub

Create `src/data/providers/massive_com.py`:

```python
import os
import logging
import requests
from typing import Optional
from src.data.providers.base import FinancialDataProvider
from src.data.models import (
    Price,
    FinancialMetrics,
    LineItem,
    InsiderTrade,
    CompanyNews,
)

logger = logging.getLogger(__name__)


class MassiveComProvider(FinancialDataProvider):
    """Massive.com API provider implementation."""
    
    # TODO: Update with actual Massive.com API URL once available
    BASE_URL = "https://api.massive.com"  # Placeholder
    
    def __init__(self, api_key: Optional[str] = None):
        super().__init__(api_key or os.environ.get("MASSIVE_API_KEY"))
    
    def _get_headers(self) -> dict:
        """Get headers for Massive.com API requests."""
        headers = {}
        if self.api_key:
            # TODO: Update auth header format per Massive.com docs
            headers["Authorization"] = f"Bearer {self.api_key}"
        return headers
    
    def get_prices(self, ticker: str, start_date: str, end_date: str) -> list[Price]:
        """Fetch price data from Massive.com."""
        # TODO: Implement once Massive.com API documentation is available
        # This is a placeholder showing the expected structure
        
        url = f"{self.BASE_URL}/quotes/historical"  # Placeholder endpoint
        params = {
            "symbol": ticker,
            "from": start_date,
            "to": end_date,
        }
        
        headers = self._get_headers()
        response = requests.get(url, headers=headers, params=params)
        
        if response.status_code != 200:
            logger.error(f"Failed to fetch prices for {ticker}: HTTP {response.status_code}")
            return []
        
        # TODO: Transform Massive.com response to our Price model
        # This will depend on their actual response format
        try:
            data = response.json()
            prices = []
            
            # Example transformation (adjust based on actual API):
            for item in data.get("results", []):
                price = Price(
                    open=item["open"],
                    close=item["close"],
                    high=item["high"],
                    low=item["low"],
                    volume=item["volume"],
                    time=item["timestamp"],
                )
                prices.append(price)
            
            return prices
        except (KeyError, ValueError, TypeError) as e:
            logger.error(f"Error parsing Massive.com price data for {ticker}: {e}")
            return []
    
    def get_financial_metrics(
        self,
        ticker: str,
        end_date: str,
        period: str = "ttm",
        limit: int = 10,
    ) -> list[FinancialMetrics]:
        """Fetch financial metrics from Massive.com."""
        # TODO: Implement once API documentation is available
        raise NotImplementedError("Massive.com financial metrics endpoint not yet implemented")
    
    def search_line_items(
        self,
        ticker: str,
        line_items: list[str],
        end_date: str,
        period: str = "ttm",
        limit: int = 10,
    ) -> list[LineItem]:
        """Search for financial statement line items."""
        # TODO: Implement once API documentation is available
        raise NotImplementedError("Massive.com line items endpoint not yet implemented")
    
    def get_insider_trades(
        self,
        ticker: str,
        end_date: str,
        start_date: Optional[str] = None,
        limit: int = 1000,
    ) -> list[InsiderTrade]:
        """Fetch insider trading data."""
        # TODO: Implement once API documentation is available
        raise NotImplementedError("Massive.com insider trades endpoint not yet implemented")
    
    def get_company_news(
        self,
        ticker: str,
        end_date: str,
        start_date: Optional[str] = None,
        limit: int = 1000,
    ) -> list[CompanyNews]:
        """Fetch company news."""
        # TODO: Implement once API documentation is available
        raise NotImplementedError("Massive.com news endpoint not yet implemented")
    
    def get_market_cap(
        self,
        ticker: str,
        end_date: str,
    ) -> Optional[float]:
        """Fetch market capitalization."""
        # TODO: Implement once API documentation is available
        raise NotImplementedError("Massive.com market cap endpoint not yet implemented")
```

### Step 1.4: Create Provider Factory

Create `src/data/providers/__init__.py`:

```python
import os
from typing import Optional
from src.data.providers.base import FinancialDataProvider
from src.data.providers.financial_datasets import FinancialDatasetsProvider
from src.data.providers.massive_com import MassiveComProvider


class ProviderFactory:
    """Factory for creating financial data providers."""
    
    _providers = {
        "financial_datasets": FinancialDatasetsProvider,
        "massive_com": MassiveComProvider,
    }
    
    @classmethod
    def create(
        cls, 
        provider_name: Optional[str] = None,
        api_key: Optional[str] = None
    ) -> FinancialDataProvider:
        """
        Create a financial data provider instance.
        
        Args:
            provider_name: Name of provider ("financial_datasets" or "massive_com")
                          If None, uses FINANCIAL_DATA_PROVIDER env variable
            api_key: Optional API key for the provider
        
        Returns:
            Instance of the requested provider
        """
        if provider_name is None:
            provider_name = os.environ.get(
                "FINANCIAL_DATA_PROVIDER", 
                "financial_datasets"
            )
        
        provider_class = cls._providers.get(provider_name.lower())
        if not provider_class:
            raise ValueError(
                f"Unknown provider: {provider_name}. "
                f"Available: {list(cls._providers.keys())}"
            )
        
        return provider_class(api_key=api_key)
    
    @classmethod
    def register_provider(cls, name: str, provider_class: type):
        """Register a new provider class."""
        cls._providers[name] = provider_class


# Convenience function
def get_provider(
    provider_name: Optional[str] = None,
    api_key: Optional[str] = None
) -> FinancialDataProvider:
    """Get a financial data provider instance."""
    return ProviderFactory.create(provider_name, api_key)
```

## Phase 2: Update API Layer (Week 2-3)

### Step 2.1: Update `src/tools/api.py`

Replace current implementation with provider-based approach:

```python
import os
import pandas as pd
from src.data.cache import get_cache
from src.data.models import Price
from src.data.providers import get_provider

# Global instances
_cache = get_cache()
_provider = None


def _get_provider():
    """Get or create the data provider instance."""
    global _provider
    if _provider is None:
        _provider = get_provider()
    return _provider


def get_prices(ticker: str, start_date: str, end_date: str, api_key: str = None) -> list[Price]:
    """Fetch price data from cache or provider."""
    cache_key = f"{ticker}_{start_date}_{end_date}"
    
    # Check cache
    if cached_data := _cache.get_prices(cache_key):
        return [Price(**price) for price in cached_data]
    
    # Fetch from provider
    provider = get_provider(api_key=api_key) if api_key else _get_provider()
    prices = provider.get_prices(ticker, start_date, end_date)
    
    if prices:
        _cache.set_prices(cache_key, [p.model_dump() for p in prices])
    
    return prices


# Similar updates for other functions...
def get_financial_metrics(
    ticker: str,
    end_date: str,
    period: str = "ttm",
    limit: int = 10,
    api_key: str = None,
):
    """Fetch financial metrics from cache or provider."""
    cache_key = f"{ticker}_{period}_{end_date}_{limit}"
    
    if cached_data := _cache.get_financial_metrics(cache_key):
        return [FinancialMetrics(**metric) for metric in cached_data]
    
    provider = get_provider(api_key=api_key) if api_key else _get_provider()
    metrics = provider.get_financial_metrics(ticker, end_date, period, limit)
    
    if metrics:
        _cache.set_financial_metrics(cache_key, [m.model_dump() for m in metrics])
    
    return metrics

# ... update all other API functions similarly
```

### Step 2.2: Update `.env.example`

```bash
# Financial Data Provider Configuration
# Options: "financial_datasets", "massive_com"
FINANCIAL_DATA_PROVIDER=financial_datasets

# For getting financial data from Financial Datasets AI
# Get your API key from https://financialdatasets.ai/
FINANCIAL_DATASETS_API_KEY=your-financial-datasets-api-key

# For getting financial data from Massive.com
# Get your API key from https://massive.com/
MASSIVE_API_KEY=your-massive-api-key
```

## Phase 3: Testing and Validation (Week 3-4)

### Step 3.1: Create Provider Tests

Create `tests/data/providers/test_providers.py`:

```python
import pytest
from src.data.providers import get_provider
from src.data.providers.financial_datasets import FinancialDatasetsProvider
from src.data.providers.massive_com import MassiveComProvider


class TestProviderFactory:
    """Test provider factory functionality."""
    
    def test_default_provider(self):
        """Test default provider is Financial Datasets."""
        provider = get_provider()
        assert isinstance(provider, FinancialDatasetsProvider)
    
    def test_financial_datasets_provider(self):
        """Test creating Financial Datasets provider."""
        provider = get_provider("financial_datasets")
        assert isinstance(provider, FinancialDatasetsProvider)
    
    def test_massive_com_provider(self):
        """Test creating Massive.com provider."""
        provider = get_provider("massive_com")
        assert isinstance(provider, MassiveComProvider)
    
    def test_invalid_provider(self):
        """Test invalid provider raises error."""
        with pytest.raises(ValueError):
            get_provider("invalid_provider")


class TestFinancialDatasetsProvider:
    """Test Financial Datasets provider."""
    
    def test_get_prices(self):
        """Test fetching prices."""
        provider = FinancialDatasetsProvider()
        prices = provider.get_prices("AAPL", "2024-01-01", "2024-01-31")
        assert isinstance(prices, list)
        # Add more assertions based on expected behavior


class TestMassiveComProvider:
    """Test Massive.com provider (when available)."""
    
    @pytest.mark.skip(reason="Massive.com API not yet available")
    def test_get_prices(self):
        """Test fetching prices from Massive.com."""
        provider = MassiveComProvider()
        prices = provider.get_prices("AAPL", "2024-01-01", "2024-01-31")
        assert isinstance(prices, list)
```

### Step 3.2: Integration Testing

Create test script `scripts/test_provider_migration.py`:

```python
#!/usr/bin/env python3
"""
Test script to compare data from both providers.
Run this to validate Massive.com API when it becomes available.
"""

import os
import logging
from datetime import datetime, timedelta
from src.data.providers import get_provider

logger = logging.getLogger(__name__)

def compare_prices(ticker: str, days: int = 30):
    """Compare price data from both providers."""
    end_date = datetime.now().strftime("%Y-%m-%d")
    start_date = (datetime.now() - timedelta(days=days)).strftime("%Y-%m-%d")
    
    print(f"\nComparing {ticker} prices from {start_date} to {end_date}")
    print("=" * 70)
    
    # Get data from Financial Datasets
    try:
        fd_provider = get_provider("financial_datasets")
        fd_prices = fd_provider.get_prices(ticker, start_date, end_date)
    except Exception as e:
        logger.error(f"Error fetching from Financial Datasets: {e}")
        fd_prices = []
    
    # Get data from Massive.com
    try:
        mc_provider = get_provider("massive_com")
        mc_prices = mc_provider.get_prices(ticker, start_date, end_date)
    except Exception as e:
        logger.error(f"Error fetching from Massive.com: {e}")
        mc_prices = []
    
    print(f"Financial Datasets: {len(fd_prices)} data points")
    print(f"Massive.com:        {len(mc_prices)} data points")
    
    if len(fd_prices) != len(mc_prices):
        print("⚠️  WARNING: Different number of data points!")
    
    # Compare sample data
    if fd_prices and mc_prices:
        print("\nSample comparison (first record):")
        print(f"FD Close:  ${fd_prices[0].close:.2f}")
        print(f"MC Close:  ${mc_prices[0].close:.2f}")
        
        diff = abs(fd_prices[0].close - mc_prices[0].close)
        pct_diff = (diff / fd_prices[0].close) * 100
        print(f"Difference: ${diff:.2f} ({pct_diff:.2f}%)")
        
        if pct_diff > 1.0:
            print("⚠️  WARNING: Price difference > 1%!")
    
    return fd_prices, mc_prices


def main():
    """Run comparison tests."""
    logging.basicConfig(level=logging.INFO)
    tickers = ["AAPL", "MSFT", "GOOGL"]
    
    for ticker in tickers:
        try:
            compare_prices(ticker)
        except Exception as e:
            logger.error(f"Error testing {ticker}: {e}")
    
    print("\n" + "=" * 70)
    print("Comparison complete!")


if __name__ == "__main__":
    main()
```

## Phase 4: Gradual Migration (Week 4-8)

### Step 4.1: Feature Flag Implementation

Update `src/data/providers/__init__.py`:

```python
def get_provider_with_fallback(
    primary_provider: str = "massive_com",
    fallback_provider: str = "financial_datasets",
    api_key: Optional[str] = None
) -> FinancialDataProvider:
    """
    Get provider with automatic fallback.
    
    This allows testing new providers while maintaining reliability.
    If primary provider fails, automatically falls back to the secondary.
    """
    import logging
    
    logger = logging.getLogger(__name__)
    
    class FallbackProvider(FinancialDataProvider):
        def __init__(self):
            self.primary = get_provider(primary_provider, api_key)
            self.fallback = get_provider(fallback_provider, api_key)
        
        def get_prices(self, ticker, start_date, end_date):
            try:
                return self.primary.get_prices(ticker, start_date, end_date)
            except Exception as e:
                logger.warning(f"Primary provider failed: {e}. Using fallback.")
                return self.fallback.get_prices(ticker, start_date, end_date)
        
        # Implement other methods similarly...
    
    return FallbackProvider()
```

### Step 4.2: Migration Checklist

Use this checklist for each endpoint migration:

- [ ] Massive.com API documentation reviewed
- [ ] Data mapping documented
- [ ] Provider method implemented
- [ ] Unit tests written
- [ ] Integration tests passing
- [ ] Data quality validated (100+ samples)
- [ ] Performance benchmarked
- [ ] Error handling tested
- [ ] Rate limiting tested
- [ ] Caching validated
- [ ] Agent integration tested
- [ ] Rollback plan documented

## Phase 5: Monitoring and Rollback (Week 8+)

### Step 5.1: Monitoring

Add logging and metrics:

```python
import logging
from datetime import datetime

logger = logging.getLogger(__name__)

class MonitoredProvider(FinancialDataProvider):
    """Wrapper that adds monitoring to any provider."""
    
    def __init__(self, provider: FinancialDataProvider):
        self.provider = provider
        self.metrics = {
            "requests": 0,
            "errors": 0,
            "cache_hits": 0,
        }
    
    def get_prices(self, ticker, start_date, end_date):
        self.metrics["requests"] += 1
        start_time = datetime.now()
        
        try:
            result = self.provider.get_prices(ticker, start_date, end_date)
            duration = (datetime.now() - start_time).total_seconds()
            logger.info(
                f"get_prices({ticker}) completed in {duration:.2f}s, "
                f"returned {len(result)} records"
            )
            return result
        except Exception as e:
            self.metrics["errors"] += 1
            logger.error(f"get_prices({ticker}) failed: {e}")
            raise
```

### Step 5.2: Rollback Procedure

If issues occur with Massive.com:

1. **Immediate Rollback:**
   ```bash
   export FINANCIAL_DATA_PROVIDER=financial_datasets
   # Restart application
   ```

2. **Gradual Rollback:**
   - Revert specific agents one at a time
   - Monitor error rates
   - Document issues for future attempts

3. **Cache Cleanup:**
   - Clear Massive.com cache entries
   - Preserve Financial Datasets cache
   - Avoid data mixing issues

## Timeline Summary

| Phase | Duration | Deliverables |
|-------|----------|-------------|
| Phase 1: Abstraction | 2 weeks | Provider interfaces, factory |
| Phase 2: API Updates | 1 week | Updated src/tools/api.py |
| Phase 3: Testing | 1 week | Test suite, validation scripts |
| Phase 4: Migration | 4 weeks | All endpoints migrated |
| Phase 5: Monitoring | Ongoing | Metrics, rollback capability |
| **Total** | **8 weeks** | **Complete migration** |

## Success Criteria

Migration is considered successful when:

1. ✅ All 6 endpoints working with Massive.com
2. ✅ Data quality matches or exceeds Financial Datasets
3. ✅ All 17 agents produce consistent results
4. ✅ Response times within acceptable limits
5. ✅ Error rate < 1%
6. ✅ Cost within budget
7. ✅ Rollback capability tested and working

## Dependencies

Before starting migration:

1. ✅ Massive.com API documentation available
2. ✅ API access credentials obtained
3. ✅ Pricing agreement finalized
4. ✅ Development resources allocated
5. ✅ Stakeholder approval obtained

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| API downtime | Automatic fallback to Financial Datasets |
| Data quality issues | Parallel comparison during migration |
| Performance degradation | Caching layer, connection pooling |
| Breaking changes | Provider abstraction isolates impact |
| Cost overruns | Usage monitoring and alerts |

## Conclusion

This migration strategy provides a safe, gradual path to swapping data providers while maintaining system reliability. The provider abstraction pattern ensures flexibility for future changes and minimizes risk through fallback mechanisms.

**Next Step:** Wait for Massive.com API documentation, then proceed with Phase 1.
