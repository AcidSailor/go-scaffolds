# Client-library API conventions — design

**Date:** 2026-07-07
**Scope:** `alor`, `finam`, `moexoptcalc`, `tinvest` client libraries + `go-scaffolds` template (+ `finamcp` as a forced consumer update).

## Goal

The four hand-written REST client libraries have drifted into two incompatible
API-design families:

- **value / flat** (`alor`, `moexoptcalc`): flat methods on `*Client`, value
  request params, value returns.
- **pointer / service-facade** (`finam`, `tinvest-REST`): service facades as
  client fields, pointer requests, pointer returns.

Errors, transport (`restkit`), and generation (oapi-codegen models-only) are
already uniform across all four. This effort converges the *API surface* onto a
single, idiomatic-Go convention and encodes it in `go-scaffolds` so future libs
inherit it.

## The convention

### Multi-resource REST client (finam, alor, tinvest-REST)

| Aspect | Rule |
|---|---|
| Service grouping | Pointer facade fields on `*Client` (`client.Orders`, `client.MarketData`). Idiom: `go-github`. |
| Facade type | Unexported struct `type ordersService struct{ c *Client }`, held as a `*ordersService` field, wired once in `NewClient`. |
| Receiver | **Pointer** — `func (s *ordersService) …`. |
| Constructor | **`NewClient(…) (*Client, error)`** + functional options. |
| Method signature | `(ctx context.Context, req XxxRequest) (*XxxResponse, error)` — **value request, pointer response**, no variadic options. |
| Method naming | Drop the redundant resource prefix only; keep the original meaning otherwise. `GetOrder`→`Get`, `GetOrders`→`List`, `PlaceOrder`→`Place`, `GetPositionsByLogin`→`PositionsByLogin`. |
| Errors / transport | Unchanged — `restkit` type aliases, `do[T]` dispatch, oapi-codegen models-only. |

### Small single-purpose client (moexoptcalc)

Stays **flat** (no facades — the API has no resource taxonomy). Adopts:

- `New` → **`NewClient`**.
- Single-object returns → **`*T`**; slice returns stay **`[]T`**.
- Value requests (already the case).
- No method rename (flat method names keep verb+noun; there is no service object
  to make the prefix redundant).

### Return-type rule (all libs)

- Single object → `*T`.
- Collection → `[]T` (not `[]*T`).

## Per-library change set

### finam (PR: finam) — closest to target
- Facade receiver: value → **pointer** (`func (s *accountsService)`; fields become `*accountsService`).
- Request: pointer → **value** (`req AccountsGetAccountRequest`).
- Responses: already `*T` — no change.
- Rename methods to drop redundant prefixes (see per-service below).
- Rename facade types `accountsServiceClient` → `accountsService` for consistency.

### finamcp (PR: finamcp) — forced consumer update
- `finamcp` consumes `finam` via `replace => ../finam`, so it breaks the moment
  finam changes. Update all call sites to the new facade field access, method
  names, and value-request signatures. Must build + test green.

### alor (PR: alor) — largest change
- Introduce service facades; flatten methods off `*Client` into services.
- Value returns → **`*T`** (single) / **`[]T`** (collections).
- Requests already value — no change.
- `NewClient` already correct.
- `ServerTime(ctx)` stays flat on `*Client` (utility).
- alormcp is version-pinned (`alor v0.1.3`) → **not** updated in this PR.

**alor taxonomy (approved):**

| Service | Method (new) | Was |
|---|---|---|
| `Orders` | `List` | `GetOrders` |
| | `Get` | `GetOrder` |
| | `PlaceMarket` | `PlaceMarketOrder` |
| | `PlaceLimit` | `PlaceLimitOrder` |
| | `ReplaceMarket` | `ReplaceMarketOrder` |
| | `ReplaceLimit` | `ReplaceLimitOrder` |
| | `Cancel` | `CancelOrder` |
| | `CancelAll` | `CancelAllOrders` |
| | `Estimate` | `EstimateOrder` |
| | `EstimateBatch` | `EstimateOrders` |
| `StopOrders` | `List` | `GetStopOrders` |
| | `Get` | `GetStopOrder` |
| | `Place` | `PlaceStopOrder` |
| | `PlaceLimit` | `PlaceStopLimitOrder` |
| | `Replace` | `ReplaceStopOrder` |
| | `ReplaceLimit` | `ReplaceStopLimitOrder` |
| `OrderGroups` | `List` | `ListOrderGroups` |
| | `Get` | `GetOrderGroup` |
| | `Create` | `CreateOrderGroup` |
| | `Update` | `UpdateOrderGroup` |
| | `Delete` | `DeleteOrderGroup` |
| `Portfolio` | `Summary` | `GetSummary` |
| | `Positions` | `GetPositions` |
| | `Position` | `GetPosition` |
| | `PositionsByLogin` | `GetPositionsByLogin` |
| | `Risk` | `GetRisk` |
| | `FortsRisk` | `GetFortsRisk` |
| | `RiskRates` | `GetRiskRates` |
| `Trades` | `Get` | `GetTrades` |
| | `Symbol` | `GetSymbolTrades` |
| | `History` | `GetTradeHistory` |
| | `SymbolHistory` | `GetSymbolTradeHistory` |
| | `All` | `GetAllTrades` |
| | `AllHistory` | `GetAllTradesHistory` |
| `MarketData` | `Search` | `SearchSecurities` |
| | `SecuritiesByExchange` | `GetSecuritiesByExchange` |
| | `Security` | `GetSecurity` |
| | `Boards` | `GetAvailableBoards` |
| | `FuturesQuote` | `GetActualFuturesQuote` |
| | `Quotes` | `GetQuotes` |
| | `CurrencyPairs` | `GetCurrencyPairs` |
| | `OrderBook` | `GetOrderBook` |
| | `History` | `GetHistory` |
| (flat) | `ServerTime` | `ServerTime` |

### moexoptcalc (PR: moexoptcalc)
- `New` → **`NewClient`**.
- Single-object returns → `*T` (e.g. `GetAsset` `(Asset, error)` → `(*Asset, error)`);
  list returns stay `[]T`; `CalculateInitialMargin` stays `(float64, error)`.
- Stays flat; value requests unchanged.
- moexoptcalcmcp version-pinned → not updated.

### tinvest (PR: tinvest) — minimal
- REST facade receiver: value → **pointer** (`func (s *usersServiceClient)`, fields `*usersServiceClient`).
- **Exempt** from the rename and value-request rules: REST is a deliberate
  parity twin of the generated gRPC transport (parity tests enforce identical
  method names and `*Request`/`*Response` shapes). Keep pointer requests and
  generated method names.
- gRPC side untouched (generated).
- tinvestmcp version-pinned → not updated.

### go-scaffolds (PR: go-scaffolds)
- Library template `{{ project_name }}.go.jinja`: `New` → `NewClient`; update `doc.go.jinja` ("Construct with `NewClient`").
- Add `templates/go-library/CONVENTIONS.md.jinja` capturing the convention table
  (facade + pointer receiver + value request + pointer response + `NewClient` +
  naming). Skeleton stays a minimal placeholder — no full restkit rewrite.
- This design doc committed under `docs/`.

## Delivery

Draft PR per repo (6): `alor`, `finam`, `finamcp`, `moexoptcalc`, `tinvest`,
`go-scaffolds`. Each branch isolated via `git worktree` under the repo's
`.claude/worktrees/`. Verification per repo: `go build ./... && go test ./...`
(and `task lint` where available). All libs are pre-1.0 → breaking changes land
in a minor bump.

## Out of scope

- Bumping/releasing version tags; updating the three version-pinned consumers
  (`alormcp`, `tinvestmcp`, `moexoptcalcmcp`) — they upgrade on their own cadence.
- Changing transport, error model, auth, or generation strategy.
- Renaming tinvest methods or changing its request pointer-ness (parity constraint).
- `[]*T` collection returns (explicitly rejected — `[]T` stays).
