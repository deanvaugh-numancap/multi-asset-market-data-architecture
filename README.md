# multi-asset-market-data-architecture
A practical reference for modelling symbols, sessions, market state, watchlists and risk context across multi-asset trading systems.
# Designing a Multi-Asset Market Data Layer

A practical reference for modelling symbols, trading sessions, market state,
watchlists and risk context across stocks, forex, crypto, commodities,
indices and metals.

**By Dean Vaugh | NumanCap**

NumanCap: https://numancap.com/

A multi-asset trading interface looks simple from the outside: search for an instrument, open a chart, add it to a watchlist, and monitor the price. Behind that workflow, however, sits a data problem that becomes increasingly complex as soon as the application needs to support stocks, forex, cryptocurrencies, commodities, indices, and metals at the same time.

Different asset classes use different naming conventions, trading schedules, price precision, market metadata, and event models. A stock such as `AAPL` behaves very differently from a currency pair such as `EUR/USD`, while a cryptocurrency can trade continuously and an equity index may follow exchange-specific sessions. If all of these instruments are represented using one overly simplistic schema, the interface eventually accumulates exceptions.

A better approach is to normalize the common parts of the market model while preserving asset-specific metadata. This is the kind of problem that becomes relevant when building a multi-market environment such as [NumanCap](https://numancap.com/about-us/), where several market categories need to coexist without pretending that they are technically identical.

## The Core Problem: One UI, Different Market Structures

Suppose an application supports the following instruments:

```text
AAPL
EUR/USD
BTC/USD
XAU/USD
S&P 500
WTI Crude Oil
```

From a user-interface perspective, all six may need a symbol, display name, current price, daily change, chart, watchlist status, and market state. That suggests a common data structure.

The problem appears when the application needs to answer more specific questions. Does the market have an exchange? Does it close overnight? What timezone should the session use? Is the instrument quoted to two decimals or five? Is the price generated from a centralized exchange, an OTC market, or another source?

A useful architecture therefore separates **shared instrument properties** from **market-specific properties**.

## Start With a Normalized Instrument Model

A base instrument object can remain intentionally small:

```json
{
  "id": "eurusd",
  "symbol": "EUR/USD",
  "display_name": "Euro / US Dollar",
  "asset_class": "forex",
  "base_currency": "EUR",
  "quote_currency": "USD",
  "price_precision": 5,
  "status": "active"
}
```

The same top-level structure could represent a stock:

```json
{
  "id": "aapl",
  "symbol": "AAPL",
  "display_name": "Apple Inc.",
  "asset_class": "stock",
  "base_currency": null,
  "quote_currency": "USD",
  "price_precision": 2,
  "status": "active"
}
```

The model does not attempt to put every possible property into the same object. Exchange metadata, contract information, trading sessions, and other attributes can live in related structures.

This keeps the common interface predictable while avoiding dozens of mostly empty fields.

## Do Not Use the Ticker as the Primary Key

A symbol looks unique until the application grows.

The same ticker can exist on different exchanges, data providers may use different symbol formats, and derivatives can have multiple contracts based on the same underlying market. Even forex pairs can appear in several naming formats:

```text
EURUSD
EUR/USD
EUR-USD
EUR_USD
```

The user-facing symbol should therefore be treated as presentation data rather than as the application's canonical identifier.

A safer pattern is:

```text
internal_id → canonical instrument
provider_symbol → external mapping
display_symbol → user interface
```

For example:

```json
{
  "instrument_id": "fx_eur_usd",
  "display_symbol": "EUR/USD",
  "provider_symbols": {
    "provider_a": "EURUSD",
    "provider_b": "EUR_USD",
    "provider_c": "EUR-USD"
  }
}
```

This mapping layer makes it possible to replace or combine data providers without rewriting the entire frontend.

## Asset Class Should Be Explicit

Inferring an asset class from the symbol format is fragile.

`BTC/USD` may resemble a forex pair syntactically, but its session structure and market behaviour are different. Gold can appear as `XAU/USD`, yet it should not automatically inherit all forex assumptions. An index may also be represented using a symbol that resembles an exchange-traded instrument.

The data model should state the classification directly:

```typescript
type AssetClass =
  | "stock"
  | "forex"
  | "crypto"
  | "commodity"
  | "index"
  | "metal";
```

This enables downstream systems to select the appropriate formatting, session logic, filters, and contextual information without guessing.

## Trading Sessions Need Their Own Model

One of the biggest differences between asset classes is time.

Cryptocurrency markets may operate continuously, while individual stock exchanges have specific trading sessions. Forex is commonly active through much of the business week, and some other markets can have breaks or different trading windows depending on the underlying structure.

Hard-coding `marketOpen = true` is therefore not enough.

A basic session object could look like:

```json
{
  "timezone": "America/New_York",
  "sessions": [
    {
      "days": ["MON", "TUE", "WED", "THU", "FRI"],
      "open": "09:30",
      "close": "16:00"
    }
  ]
}
```

For a continuously active market:

```json
{
  "timezone": "UTC",
  "sessions": [
    {
      "days": ["MON", "TUE", "WED", "THU", "FRI", "SAT", "SUN"],
      "open": "00:00",
      "close": "23:59"
    }
  ]
}
```

In production, holidays, early closes, maintenance windows, daylight-saving changes, and exceptional closures also need to be considered.

The important architectural decision is to make session state a service or model of its own rather than scattering time logic throughout UI components.

## Price Precision Is Market Data, Not Frontend Styling

A common UI mistake is formatting every price to two decimal places.

That may look reasonable for many stocks, but it breaks quickly for forex, crypto, and instruments whose useful price increments are much smaller.

Instead of doing this:

```javascript
price.toFixed(2)
```

the formatter should use instrument metadata:

```javascript
function formatPrice(price, instrument) {
  return price.toFixed(instrument.price_precision);
}
```

That same metadata layer can eventually support tick size, minimum order increments, and other market-specific rules.

Presentation becomes more reliable when formatting rules originate in domain data rather than being invented by individual components.

## Separate Instrument Identity From Live Market State

An instrument's identity changes rarely. Its current market state may change many times per second.

These two concerns should not be stored or delivered in exactly the same way.

Static or slow-changing metadata might include:

```json
{
  "instrument_id": "stock_aapl",
  "symbol": "AAPL",
  "asset_class": "stock",
  "exchange": "NASDAQ",
  "currency": "USD"
}
```

Live state might look like:

```json
{
  "instrument_id": "stock_aapl",
  "bid": 271.12,
  "ask": 271.16,
  "last": 271.14,
  "change_percent": 0.82,
  "timestamp": "2026-10-30T14:31:17Z"
}
```

Keeping these layers separate makes caching, streaming, persistence, and frontend updates easier to reason about.

There is little value in retransmitting an instrument's full metadata every time its price changes.

## Watchlists Should Store References, Not Copies

A watchlist is conceptually simple, but poor data modelling can create synchronization problems.

Avoid storing a complete instrument snapshot inside every watchlist item. Instead, store the canonical instrument ID and user-specific metadata:

```json
{
  "watchlist_id": "macro",
  "items": [
    {
      "instrument_id": "fx_eur_usd",
      "position": 1
    },
    {
      "instrument_id": "metal_xau_usd",
      "position": 2
    }
  ]
}
```

The application can resolve current instrument data from the central instrument service.

This prevents old names, prices, classifications, or metadata from becoming embedded in multiple user records.

## A Watchlist Can Also Be a Research Tool

A useful multi-asset watchlist is not necessarily a list of markets the user intends to trade.

It can represent a market hypothesis.

For example:

```text
Macro Watchlist

S&P 500
NASDAQ 100
EUR/USD
USD/CAD
Gold
WTI Crude Oil
Bitcoin
```

Together, these instruments can provide information about equity sentiment, dollar strength, Canadian macro conditions, precious metals, energy, and crypto risk appetite.

This is one reason multi-asset interfaces can provide more value than simply increasing the number of searchable instruments. A well-designed watchlist allows users to compare related markets without forcing them to open positions in each one.

## Search Needs Normalization Too

Market search becomes complicated when users enter:

```text
bitcoin
BTC
btc usd
BTC/USD
gold
xau
XAUUSD
```

All of these can be legitimate attempts to find an instrument.

A useful search index therefore needs more than an exact symbol match. It can combine normalized symbols, display names, aliases, asset-class terms, and provider-independent identifiers.

An indexed search document might contain:

```json
{
  "instrument_id": "crypto_btc_usd",
  "symbol": "BTC/USD",
  "name": "Bitcoin / US Dollar",
  "aliases": [
    "BTC",
    "Bitcoin",
    "BTCUSD",
    "BTC USD"
  ],
  "asset_class": "crypto"
}
```

The UI can then return the canonical instrument regardless of how the user describes it.

## Market Context Should Be Part of the Interface

Showing only the latest price turns a trading dashboard into an expensive ticker board.

Useful market interfaces provide context. That can include the current session, percentage change, recent range, volatility state, relevant event information, or comparisons with related markets.

The exact information depends on the asset class.

For stocks, earnings dates may matter. For forex, central-bank and macroeconomic events may be more relevant. Commodity analysis may require inventory or supply context, while crypto has its own continuously active market structure.

The architecture therefore benefits from a contextual layer:

```text
Instrument
   ↓
Live Market Data
   ↓
Session State
   ↓
Market Context
   ↓
User View
```

This allows the frontend to assemble an experience appropriate to the market rather than presenting every asset class as the same generic chart.

## Risk Context Should Travel With the Instrument

One of the more important design principles in financial software is that risk information should not be hidden several navigation levels away from the trading action.

If an instrument has unusual volatility, leverage conditions, market closures, contract expiry, or other relevant characteristics, the interface should have a way to surface that context close to the point where it matters.

This does not mean filling the screen with warnings. It means designing the data model so risk-related information is available when the application needs it.

For example:

```json
{
  "instrument_id": "example_market",
  "risk_context": {
    "leverage_applicable": true,
    "market_hours_restricted": true,
    "contract_expiry": null,
    "volatility_class": "high"
  }
}
```

The UI can decide how and when to display those properties.

A system that never models risk context cannot communicate it intelligently later.

## Correlation Is a Portfolio Problem

Supporting several asset classes creates another challenge: different symbols may represent similar economic exposure.

A user could follow or trade technology stocks, a major equity index, and a cryptocurrency. Technically, these are different instruments. Under certain market conditions, however, all three may respond to the same change in liquidity or risk sentiment.

Likewise, oil, the Canadian dollar, and energy equities can sometimes share an important macro driver.

A sophisticated platform should therefore think beyond instrument-level risk and eventually support account-level or portfolio-level context.

Conceptually:

```text
Position A ─┐
Position B ─┼─→ Shared macro factor → Portfolio exposure
Position C ─┘
```

Building a perfect real-time correlation engine is not required on day one. Designing the system so positions can later be grouped by asset class, currency, sector, or broader factor is already valuable.

## Avoid Building Business Logic Into React Components

A frontend component should not decide whether a market is open, how a ticker maps to a provider, or how a futures contract expires.

Those are domain rules.

A cleaner architecture might look like:

```text
Data Providers
      ↓
Normalization Layer
      ↓
Instrument Service
      ↓
Market State Service
      ↓
API / Streaming Layer
      ↓
Frontend
```

The normalization layer converts provider-specific representations into a common internal model. The instrument service owns identity and metadata, while the market-state layer handles frequently changing information.

The frontend receives an already coherent representation.

That keeps UI code focused on rendering and interaction rather than financial-domain interpretation.

## Provider Independence Is Worth Designing Early

Market-data integrations often begin with a single provider. That can make provider-specific symbols and schemas feel harmless.

The problem appears later when another data source is introduced for redundancy, additional markets, or better coverage.

Without a normalization layer, the provider's model effectively becomes the application's domain model. Replacing the provider then becomes a rewrite.

Instead, introduce an adapter:

```typescript
interface MarketDataAdapter {
  normalizeInstrument(raw: unknown): Instrument;
  normalizeQuote(raw: unknown): Quote;
  normalizeCandle(raw: unknown): Candle;
}
```

Each provider implements the same internal interface.

That does not eliminate differences between vendors, but it confines those differences to a predictable boundary.

## Cache According to Data Velocity

Not every financial-data object has the same freshness requirements.

Instrument names and classifications may change rarely. Session state changes predictably. Quotes can update continuously, while historical candles may become immutable after a period closes.

Treating all of these objects with the same cache policy is inefficient.

A simple classification might be:

| Data | Typical Behaviour |
|---|---|
| Instrument metadata | Slow-changing |
| Trading schedule | Predictable / occasional changes |
| Live quotes | High-frequency |
| Current candle | Frequently changing |
| Closed historical candle | Mostly immutable |
| User watchlist | Event-driven |
| Market context | Depends on source |

This separation makes infrastructure easier to scale because high-frequency data does not invalidate everything around it.

## Error States Matter in Financial Interfaces

Displaying stale data as if it were current is worse than showing no data.

Market applications should distinguish between:

```text
Loading
Live
Delayed
Stale
Unavailable
Market Closed
Error
```

These are not merely UI labels. They represent different states in the data pipeline.

A quote can be technically valid but old. A market can be functioning normally while currently closed. A provider can stop sending updates even though the last known value remains in the cache.

The application should preserve enough metadata to distinguish these situations.

A quote object might therefore include:

```json
{
  "instrument_id": "fx_eur_usd",
  "price": 1.16245,
  "timestamp": "2026-10-30T15:01:08Z",
  "source_status": "live"
}
```

Freshness can then be evaluated separately from the price itself.

## The Interface Should Reveal Uncertainty

Financial software often looks more precise than the underlying information actually is.

A price can change between the moment it is transmitted and the moment the user sees it. A displayed spread may change before an order is submitted. A market relationship that held yesterday may disappear today.

Good interfaces avoid turning dynamic data into false certainty.

That can mean showing timestamps, communicating delayed data clearly, distinguishing indicative information from executable conditions, and avoiding UI patterns that imply a forecast is guaranteed.

The engineering principle is simple: **do not hide uncertainty that exists in the underlying system.**

## How This Connects to NumanCap

From a product perspective, a multi-market platform such as **NumanCap** has to solve a broader problem than displaying a collection of instruments. Stocks, forex, cryptocurrencies, commodities, indices, and metals need enough normalization to feel consistent while retaining the characteristics that make each market different.

That balance is what makes multi-asset architecture interesting. The user benefits from a unified interface, but the underlying engineering should not erase market-specific behaviour simply to make the schema look cleaner.

The strongest design is therefore not “one model for everything.” It is **one coherent abstraction with explicit extensions for differences that actually matter**.

## A Practical Architecture

A simplified multi-asset system could be represented as:

```text
┌───────────────────────────────┐
│       External Providers      │
│ quotes / candles / metadata   │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│       Provider Adapters       │
│ symbol + schema normalization │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│      Instrument Registry      │
│ identity / asset class / meta │
└───────────────┬───────────────┘
                │
       ┌────────┴────────┐
       ▼                 ▼
┌─────────────┐   ┌─────────────┐
│ Live Market │   │ Market      │
│ State       │   │ Context     │
└──────┬──────┘   └──────┬──────┘
       └────────┬─────────┘
                ▼
┌───────────────────────────────┐
│        API / WebSocket        │
└───────────────┬───────────────┘
                ▼
┌───────────────────────────────┐
│       Web / Mobile UI         │
│ search / charts / watchlists  │
└───────────────────────────────┘
```

The exact infrastructure can vary enormously, but the boundaries are more important than the technology choices. Provider-specific complexity remains near the integration layer, normalized market concepts remain in the domain layer, and UI components consume a stable representation.

## Repository Checklist

Before calling a multi-asset market-data model complete, it is worth checking whether the architecture can answer the following questions:

- Can two providers use different symbols for the same instrument?
- Can two exchanges contain instruments with the same ticker?
- Can each asset class define its own price precision?
- Can markets have different sessions and timezones?
- Can the application distinguish closed, stale, and unavailable data?
- Can users search using aliases rather than exact symbols?
- Can watchlists reference canonical instruments?
- Can risk-related metadata be attached to an instrument?
- Can the system support a second data provider without changing the frontend?
- Can portfolio-level exposure eventually be grouped across asset classes?

If several answers are “no,” the model may still work for the current prototype, but the missing abstraction will probably appear again as the product grows.

## Final Thoughts

Multi-asset financial applications are fundamentally normalization systems. The frontend may show one search box and one watchlist, but the markets behind them use different symbols, sessions, price formats, trading conventions, and risk characteristics.

Trying to remove every difference creates fragile abstractions. Treating every asset class as a completely separate product creates duplication. The useful middle ground is a stable shared model for common concepts combined with explicit market-specific metadata where the differences matter.

For NumanCap and other multi-market products, that architecture supports something more important than technical elegance: it allows users to move between markets without losing the context that makes each market understandable.

A good multi-asset interface should feel unified. Its data model should know that the markets underneath it are not.
