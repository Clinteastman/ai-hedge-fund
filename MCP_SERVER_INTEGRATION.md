# MCP Server Integration for API Access

**Purpose:** This document explores using Model Context Protocol (MCP) servers to access external APIs like Massive.com that may be blocked by network restrictions.

## What is MCP?

Model Context Protocol (MCP) is an open protocol that enables secure connections between AI applications and external data sources through standardized servers. MCP servers act as intermediaries that can:

1. Access external APIs and web resources
2. Provide data to AI assistants in a controlled manner
3. Handle authentication and rate limiting
4. Normalize data formats

## Use Case: Accessing Massive.com API

### Current Problem

- The Massive.com API documentation (https://massive.com/docs/rest/quickstart) is blocked by network restrictions
- Cannot programmatically access the API to validate endpoints and data formats
- Manual review required, slowing down investigation

### MCP Solution

An MCP server could:
1. Run in an environment with access to massive.com
2. Fetch and parse API documentation
3. Make test API requests to Massive.com
4. Return formatted results to the AI assistant
5. Cache responses to avoid repeated requests

## Potential MCP Servers for This Project

### 1. Web Browsing MCP Server

**Purpose:** Access web documentation

**Capabilities:**
- Fetch web pages (including Massive.com docs)
- Parse HTML content
- Extract API specifications
- Follow links and navigate documentation

**Example Usage:**
```python
# Hypothetical MCP server call
documentation = mcp_fetch(
    url="https://massive.com/docs/rest/quickstart",
    extract="api_endpoints"
)
```

### 2. HTTP API MCP Server

**Purpose:** Test API endpoints directly

**Capabilities:**
- Make HTTP requests to external APIs
- Handle authentication (API keys, OAuth)
- Parse JSON/XML responses
- Respect rate limits

**Example Usage:**
```python
# Test Massive.com API directly
response = mcp_api_request(
    url="https://api.massive.com/v1/prices",
    method="GET",
    params={"symbol": "AAPL", "date": "2024-01-01"},
    auth_header="Bearer API_KEY"
)
```

### 3. Financial Data MCP Server (Custom)

**Purpose:** Unified interface for multiple financial data providers

**Capabilities:**
- Abstract multiple financial data APIs (Financial Datasets, Massive, others)
- Provide consistent data format regardless of source
- Handle provider-specific authentication
- Implement fallback logic

**Example Usage:**
```python
# Get prices from any provider
prices = mcp_financial_data(
    provider="massive",  # or "financial_datasets"
    data_type="prices",
    ticker="AAPL",
    start_date="2024-01-01",
    end_date="2024-12-31"
)
```

## Implementation Options

### Option 1: Use Existing MCP Servers

Several open-source MCP servers exist that could help:

**A. Fetch MCP Server** (for web content)
- Repository: Available in MCP server registry
- Purpose: Fetch web pages and extract content
- Setup: Install and configure with allowed domains

**B. Puppeteer MCP Server** (for complex web apps)
- Repository: Available in MCP server registry
- Purpose: Browse websites with JavaScript support
- Setup: Requires Node.js and Puppeteer

### Option 2: Create Custom MCP Server

Build a dedicated MCP server for financial data access:

**File: `mcp_servers/financial_api_server.py`**

```python
#!/usr/bin/env python3
"""
MCP Server for Financial Data APIs
Provides access to multiple financial data providers
"""

import asyncio
import json
from typing import Any, Dict
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationOptions
import mcp.types as types
import httpx

# Initialize MCP server
server = Server("financial-api-server")

# Supported providers
PROVIDERS = {
    "financial_datasets": {
        "base_url": "https://api.financialdatasets.ai",
        "auth_header": "X-API-KEY"
    },
    "massive": {
        "base_url": "https://api.massive.com",  # TBD
        "auth_header": "Authorization"
    }
}


@server.list_tools()
async def handle_list_tools() -> list[types.Tool]:
    """List available tools."""
    return [
        types.Tool(
            name="fetch_api_docs",
            description="Fetch API documentation from Massive.com",
            inputSchema={
                "type": "object",
                "properties": {
                    "url": {
                        "type": "string",
                        "description": "Documentation URL to fetch"
                    }
                },
                "required": ["url"]
            }
        ),
        types.Tool(
            name="get_stock_prices",
            description="Get stock prices from any supported provider",
            inputSchema={
                "type": "object",
                "properties": {
                    "provider": {
                        "type": "string",
                        "enum": ["financial_datasets", "massive"],
                        "description": "Data provider to use"
                    },
                    "ticker": {
                        "type": "string",
                        "description": "Stock ticker symbol"
                    },
                    "start_date": {
                        "type": "string",
                        "description": "Start date (YYYY-MM-DD)"
                    },
                    "end_date": {
                        "type": "string",
                        "description": "End date (YYYY-MM-DD)"
                    }
                },
                "required": ["provider", "ticker", "start_date", "end_date"]
            }
        ),
        types.Tool(
            name="compare_providers",
            description="Compare data quality between providers",
            inputSchema={
                "type": "object",
                "properties": {
                    "ticker": {"type": "string"},
                    "date": {"type": "string"}
                },
                "required": ["ticker", "date"]
            }
        )
    ]


@server.call_tool()
async def handle_call_tool(
    name: str, 
    arguments: dict | None
) -> list[types.TextContent | types.ImageContent | types.EmbeddedResource]:
    """Handle tool calls."""
    
    if name == "fetch_api_docs":
        url = arguments.get("url")
        
        async with httpx.AsyncClient() as client:
            try:
                response = await client.get(url, timeout=30.0)
                response.raise_for_status()
                
                return [
                    types.TextContent(
                        type="text",
                        text=f"API Documentation from {url}:\n\n{response.text[:5000]}"
                    )
                ]
            except Exception as e:
                return [
                    types.TextContent(
                        type="text",
                        text=f"Error fetching documentation: {str(e)}"
                    )
                ]
    
    elif name == "get_stock_prices":
        provider = arguments.get("provider")
        ticker = arguments.get("ticker")
        start_date = arguments.get("start_date")
        end_date = arguments.get("end_date")
        
        # Get provider configuration
        provider_config = PROVIDERS.get(provider)
        if not provider_config:
            return [types.TextContent(
                type="text",
                text=f"Unknown provider: {provider}"
            )]
        
        # Build API URL (provider-specific)
        if provider == "financial_datasets":
            url = f"{provider_config['base_url']}/prices/?ticker={ticker}&start_date={start_date}&end_date={end_date}&interval=day&interval_multiplier=1"
        elif provider == "massive":
            # TBD - Update when Massive API format is known
            url = f"{provider_config['base_url']}/quotes/historical?symbol={ticker}&from={start_date}&to={end_date}"
        
        # Make API request
        async with httpx.AsyncClient() as client:
            try:
                # Add authentication if available
                headers = {}
                api_key = os.environ.get(f"{provider.upper()}_API_KEY")
                if api_key:
                    if provider == "massive":
                        headers[provider_config['auth_header']] = f"Bearer {api_key}"
                    else:
                        headers[provider_config['auth_header']] = api_key
                
                response = await client.get(url, headers=headers, timeout=30.0)
                response.raise_for_status()
                
                data = response.json()
                
                return [
                    types.TextContent(
                        type="text",
                        text=f"Stock prices for {ticker} from {provider}:\n\n{json.dumps(data, indent=2)}"
                    )
                ]
            except Exception as e:
                return [
                    types.TextContent(
                        type="text",
                        text=f"Error fetching prices: {str(e)}"
                    )
                ]
    
    elif name == "compare_providers":
        ticker = arguments.get("ticker")
        date = arguments.get("date")
        
        # Fetch from both providers
        results = {}
        for provider in ["financial_datasets", "massive"]:
            try:
                # Call get_stock_prices for each provider
                prices = await handle_call_tool(
                    "get_stock_prices",
                    {
                        "provider": provider,
                        "ticker": ticker,
                        "start_date": date,
                        "end_date": date
                    }
                )
                results[provider] = prices[0].text if prices else "No data"
            except Exception as e:
                results[provider] = f"Error: {str(e)}"
        
        comparison = f"""
Provider Comparison for {ticker} on {date}:

Financial Datasets:
{results.get('financial_datasets', 'N/A')}

Massive.com:
{results.get('massive', 'N/A')}
"""
        
        return [types.TextContent(type="text", text=comparison)]
    
    else:
        raise ValueError(f"Unknown tool: {name}")


async def main():
    """Main entry point."""
    from mcp.server.stdio import stdio_server
    
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            InitializationOptions(
                server_name="financial-api-server",
                server_version="0.1.0",
                capabilities=server.get_capabilities(
                    notification_options=NotificationOptions(),
                    experimental_capabilities={},
                ),
            ),
        )


if __name__ == "__main__":
    import os
    asyncio.run(main())
```

**File: `pyproject.toml` (update)**

```toml
[project.optional-dependencies]
mcp = [
    "mcp>=0.1.0",
    "httpx>=0.24.0"
]
```

### Option 3: Configure Environment for MCP

**Environment Setup:**

1. **Install MCP SDK**:
```bash
pip install mcp httpx
```

2. **Configure MCP Server** in `~/.config/mcp/servers.json`:
```json
{
  "financial-api-server": {
    "command": "python",
    "args": ["/path/to/mcp_servers/financial_api_server.py"],
    "env": {
      "FINANCIAL_DATASETS_API_KEY": "${FINANCIAL_DATASETS_API_KEY}",
      "MASSIVE_API_KEY": "${MASSIVE_API_KEY}"
    }
  }
}
```

3. **Use in Code**:
```python
from mcp import ClientSession

async with ClientSession("financial-api-server") as session:
    # Fetch Massive.com documentation
    docs = await session.call_tool(
        "fetch_api_docs",
        {"url": "https://massive.com/docs/rest/quickstart"}
    )
    
    # Get stock prices
    prices = await session.call_tool(
        "get_stock_prices",
        {
            "provider": "massive",
            "ticker": "AAPL",
            "start_date": "2024-01-01",
            "end_date": "2024-01-31"
        }
    )
```

## Benefits for This Investigation

### Immediate Benefits

1. **Access Blocked Documentation**: Fetch Massive.com docs through MCP server running in unrestricted environment
2. **Test APIs Directly**: Make test requests to validate endpoint behavior
3. **Compare Providers**: Side-by-side comparison of Financial Datasets vs Massive
4. **Automated Analysis**: Programmatically analyze API responses

### Long-term Benefits

1. **Provider Abstraction**: MCP server becomes the abstraction layer
2. **Multi-Provider Support**: Easy to add new providers
3. **Centralized Auth**: Manage API keys in one place
4. **Rate Limiting**: MCP server handles rate limits across all tools
5. **Caching**: Built-in caching reduces redundant API calls

## Implementation Plan

### Phase 1: Setup (1-2 days)

1. Install MCP SDK in development environment
2. Create basic financial API MCP server
3. Test with Financial Datasets API (already accessible)
4. Verify MCP server can run in unrestricted environment

### Phase 2: Massive.com Integration (2-3 days)

1. Deploy MCP server to environment with massive.com access
2. Fetch and parse Massive.com documentation
3. Implement Massive.com endpoints in MCP server
4. Test data retrieval and formatting

### Phase 3: Migration Support (1-2 weeks)

1. Update migration strategy to use MCP server
2. Modify `src/tools/api.py` to support MCP provider
3. Create comprehensive tests
4. Document MCP server usage

## Alternative: Quick MCP Server for Documentation Only

If full implementation is too complex, create a minimal MCP server just for fetching docs:

**File: `scripts/fetch_massive_docs.py`**

```python
#!/usr/bin/env python3
"""
Simple script to fetch Massive.com documentation
Run this in an environment with network access to massive.com
"""

import httpx
import json
from bs4 import BeautifulSoup


def fetch_massive_docs():
    """Fetch and parse Massive.com API documentation."""
    
    urls = [
        "https://massive.com/docs/rest/quickstart",
        "https://massive.com/docs/rest/authentication",
        "https://massive.com/docs/rest/endpoints",
        # Add more doc URLs as discovered
    ]
    
    documentation = {}
    
    for url in urls:
        try:
            with httpx.Client() as client:
                response = client.get(url, timeout=30.0)
                response.raise_for_status()
                
                # Parse HTML
                soup = BeautifulSoup(response.text, 'html.parser')
                
                # Extract relevant content
                documentation[url] = {
                    "title": soup.title.string if soup.title else "Unknown",
                    "content": soup.get_text()[:5000],  # First 5000 chars
                    "code_examples": [code.get_text() for code in soup.find_all('code')]
                }
                
                print(f"✓ Fetched: {url}")
        except Exception as e:
            print(f"✗ Error fetching {url}: {e}")
            documentation[url] = {"error": str(e)}
    
    # Save to file
    output_file = "massive_api_documentation.json"
    with open(output_file, 'w') as f:
        json.dump(documentation, f, indent=2)
    
    print(f"\n✓ Documentation saved to {output_file}")
    return documentation


if __name__ == "__main__":
    fetch_massive_docs()
```

**Usage:**
1. Run this script on a machine with access to massive.com
2. Copy the resulting `massive_api_documentation.json` file
3. Add to repository for team review
4. Use content to update migration documents

## Recommendation

**For This Investigation:**

1. **Immediate**: Use the simple documentation fetcher script above
   - Minimal setup required
   - Can run on any machine with network access
   - Results can be committed to repository

2. **Short-term**: Consider building custom MCP server
   - Provides ongoing access for testing
   - Useful for comparing providers
   - Supports migration validation

3. **Long-term**: Integrate MCP as provider abstraction
   - MCP server becomes the standard way to access financial data
   - All agents use MCP instead of direct API calls
   - Easy to add/swap providers

## Next Steps

1. **Determine if MCP infrastructure is available** in your environment
2. **If yes**: Set up financial API MCP server following Phase 1 above
3. **If no**: Use simple documentation fetcher script to get Massive.com docs
4. **Then**: Update migration documents with actual API specifications

---

**Related Documents:**
- [MIGRATION_STRATEGY.md](./MIGRATION_STRATEGY.md) - Can be updated to use MCP
- [MASSIVE_API_ADDENDUM.md](./MASSIVE_API_ADDENDUM.md) - Lists what to extract from docs

**Status:** MCP integration is **optional but recommended**  
**Effort:** 1-2 weeks for full implementation, or 1 day for simple doc fetcher  
**Benefit:** Significantly simplifies API access and testing
