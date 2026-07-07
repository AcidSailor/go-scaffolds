# Client-library API conventions convergence — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Converge the four client libraries (`alor`, `finam`, `moexoptcalc`, `tinvest`) onto one idiomatic-Go API convention and encode it in the `go-scaffolds` template.

**Architecture:** Each repo is an independent Go module → one isolated `git worktree` + one draft PR each. The refactors are mechanical (rename methods, group into service facades, flip receiver/pointer conventions) verified by the existing test suite compiling and passing against the new API. Generated files (`models.gen.go`, `grpc/pb/**`) are never edited.

**Tech Stack:** Go 1.26, `github.com/acidsailor/restkit`, oapi-codegen v2 (models-only), Taskfile, golangci-lint.

## Global Constraints

- **Never edit generated files:** `models.gen.go`, `rest/models.gen.go`, `grpc/pb/**`. Response DTOs keep their generated names.
- **Return rule:** single object → `*T`; collection → `[]T` (never `[]*T`); scalars unchanged.
- **Request struct naming:** `<Service><Method>Request` (faceted libs) / `<Method>Request` (flat lib), using the *renamed* method; suffix unified on `Request` (replaces `Params`). Methods with no input have no request struct.
- **Facade pattern (faceted libs):** `type xxxService struct{ c *Client }`, held as an exported `*xxxService` field on `Client`, wired once in `NewClient`; **pointer receivers** `func (s *xxxService)`.
- **Constructor:** exported name `NewClient`.
- **Method naming:** drop the redundant resource prefix only (`GetOrder`→`Get`, `GetOrders`→`List`); keep original meaning otherwise.
- **Do NOT touch** version-pinned consumers `alormcp`, `tinvestmcp`, `moexoptcalcmcp`.
- **Per-repo verification gate:** `go build ./... && go test ./...` (plus `task lint` if a `taskfile.yml`/`.golangci.yaml` is present) must be green before commit.
- **Delivery per repo:** worktree under `<repo>/.claude/worktrees/api-conventions`, branch `feat/api-conventions`, then `gh pr create --draft`. Pre-1.0 → breaking changes are fine.

## Execution order & parallelism

- **Independent, run in parallel:** `go-scaffolds` (Task 1), `moexoptcalc` (Task 2), `tinvest` (Task 3), `alor` (Task 4), `finam` (Task 5).
- **Sequential:** `finamcp` (Task 6) **must run after** `finam` (Task 5) — it consumes finam via `replace => ../finam` and won't compile until finam's new API exists.

---

### Task 1: go-scaffolds — template + conventions doc

**Repo:** `go-scaffolds` (branch `feat/client-lib-conventions` already exists with the spec+plan). Land the template change on the same branch.

**Files:**
- Modify: `templates/go-library/{{ project_name }}.go.jinja` (rename `New` → `NewClient`)
- Modify: `templates/go-library/doc.go.jinja` ("Construct with `New`" → "`NewClient`")
- Modify: `templates/go-library/{{ project_name }}_test.go.jinja` (any `New(` call → `NewClient(`)
- Create: `templates/go-library/CONVENTIONS.md.jinja`

- [ ] **Step 1: Rename constructor in the template**

In `{{ project_name }}.go.jinja`, change the func and its doc comment:
```go
// NewClient constructs a Client. The default *http.Client is instrumented with
// otelhttp so requests produce spans when a TracerProvider is configured.
func NewClient(baseURL string, opts ...Option) *Client {
```

- [ ] **Step 2: Update doc.go.jinja**

Change the sentence to: `// client. Construct with NewClient; the default transport is otelhttp-instrumented.`

- [ ] **Step 3: Update the test template**

In `{{ project_name }}_test.go.jinja`, replace any `New(` invocation with `NewClient(`. Verify with:
Run: `grep -n "New(" "templates/go-library/{{ project_name }}_test.go.jinja"`
Expected: only `NewClient(` remains.

- [ ] **Step 4: Create CONVENTIONS.md.jinja**

```markdown
# API design conventions

This library follows the shared conventions for `acidsailor` Go client libraries.

## Constructor
`NewClient(...) (*Client, error)` with functional options (`Option` / `ClientOption`).

## Service grouping (multi-resource clients)
Group methods into service facades exposed as pointer fields on `*Client`:
`client.Orders.Get(ctx, req)`. Facade type is an unexported `type ordersService
struct{ c *Client }` held as a `*ordersService` field, wired once in `NewClient`,
with **pointer receivers**. Small single-purpose clients stay flat.

## Method signatures
`(ctx context.Context, req XxxRequest) (*XxxResponse, error)`:
value request, pointer response, no variadic options.

## Returns
Single object → `*T`; collection → `[]T`.

## Naming
- Methods drop the redundant resource prefix: `GetOrder`→`Get`, `GetOrders`→`List`.
- Request structs: `<Service><Method>Request` (faceted) / `<Method>Request` (flat).
- Response DTOs keep their generated names.

## Errors / transport
`restkit` type aliases (`ResponseError`/`RequestError`/`ConfigError`) + `Op*`
constants; no sentinels. Models generated by oapi-codegen (models-only).
```

- [ ] **Step 5: Verify template still renders (smoke)**

Run: `grep -rn "func New(" templates/go-library/ ; echo "exit: $?"`
Expected: no matches (exit 1) — all constructors are `NewClient`.

- [ ] **Step 6: Commit**

```bash
git add templates/go-library/
git commit -m "feat(go-library): adopt NewClient + add API CONVENTIONS doc"
```

- [ ] **Step 7: Push + draft PR**

```bash
git push -u origin feat/client-lib-conventions
gh pr create --draft --title "feat: client-library API conventions (spec + template)" \
  --body "$(cat <<'EOF'
Adds the client-library API conventions design spec + implementation plan, and
updates the go-library template (`NewClient`, `CONVENTIONS.md`).

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

---

### Task 2: moexoptcalc — NewClient + pointer single-returns + Params→Request

**Repo:** `moexoptcalc`. Stays **flat** (no facades). Value requests unchanged.

**Files:**
- Modify: `client.go` (`New`→`NewClient`; option type `ClientOption` unchanged; single-object return types → `*T`; call sites `do[T]`→`do[*T]`)
- Modify: `params.go` (rename every `XxxParams` struct → `XxxRequest`)
- Modify: `client_test.go`, `client_transport_test.go`, `date_test.go`, `datetime_test.go` (update `New(`→`NewClient(`, `*Params`→`*Request`, and pointer-return assertions)

**Interfaces — produced (new public surface):**
- `func NewClient(endpoint string, opts ...ClientOption) (*Client, error)`
- Pointer single-returns:
  - `GetAsset(ctx, GetAssetRequest) (*Asset, error)`
  - `GetOption(ctx, GetOptionRequest) (*OptionBrief, error)`
  - `GetOptionSeries(ctx, GetOptionSeriesRequest) (*OptionSeries, error)`
  - `GetOptionBoard(ctx, GetOptionBoardRequest) (*OptionBoard, error)`
  - `CalculatePortfolio(ctx, OptionPortfolio) (*CalculatedPortfolio, error)`
  - `CalculatePortfolioGraph(ctx, OptionPortfolio, CalculatePortfolioGraphRequest) (*IndicatorGraph, error)`
- Unchanged returns: `ListAssets`/`ListFutures`/`ListOptions`/`ListOptionSeries`/`ListSeriesOptions` → `[]T`; `GetVolatilityGraph` → `[]VolatilityGraphPoint`; `CalculateInitialMargin` → `float64`.
- Request renames (params.go): `ListAssetsParams`→`ListAssetsRequest`, `GetAssetParams`→`GetAssetRequest`, `ListFuturesParams`→`ListFuturesRequest`, `ListOptionsParams`→`ListOptionsRequest`, `GetOptionParams`→`GetOptionRequest`, `ListOptionSeriesParams`→`ListOptionSeriesRequest`, `GetOptionSeriesParams`→`GetOptionSeriesRequest`, `ListSeriesOptionsParams`→`ListSeriesOptionsRequest`, `GetOptionBoardParams`→`GetOptionBoardRequest`, `GetVolatilityGraphParams`→`GetVolatilityGraphRequest`, `CalculatePortfolioGraphParams`→`CalculatePortfolioGraphRequest`.

- [ ] **Step 1: Rename constructor**

In `client.go`, rename `func New(` → `func NewClient(` and update its doc comment. Leave `ClientOption`/`WithHTTPClient` as-is.

- [ ] **Step 2: Rename param structs → Request (params.go)**

Apply the 11 renames above to the `type X struct` declarations and their doc comments.

- [ ] **Step 3: Update method signatures & dispatch (client.go)**

For each single-object method, change the return type to `*T` and its `do`/`restkit.Do` type argument to the pointer form. Example (GetAsset):
```go
func (c *Client) GetAsset(ctx context.Context, params GetAssetRequest) (*Asset, error) {
	// ...build query/path unchanged...
	return restkit.Do[*Asset](ctx, c.rkClient, http.MethodGet, path, nil)
}
```
Update the `params <Old>Params` parameter types to the new `<New>Request` names in every method. Leave slice-returning and `float64` methods' return types unchanged (only their param type names change).

- [ ] **Step 4: Update tests**

In the four test files, replace `New(` → `NewClient(`, every `*Params` type name → `*Request`, and adjust assertions for single-object methods to expect `*T` (e.g. `got.Symbol` still works via pointer; nil-check where a value was previously compared).

- [ ] **Step 5: Build + test**

Run: `go build ./... && go test ./...`
Expected: PASS. If `restkit.Do[*Asset]` fails to decode, confirm restkit supports pointer `T` (finam relies on this — it does); do not change restkit.

- [ ] **Step 6: Lint**

Run: `task lint 2>/dev/null || golangci-lint run`
Expected: clean.

- [ ] **Step 7: Commit + worktree + PR**

```bash
git worktree add -b feat/api-conventions .claude/worktrees/api-conventions origin/main
# (perform the edits inside the worktree, then:)
git add -A && git commit -m "feat!: adopt NewClient, pointer single-returns, Request-suffixed params"
git push -u origin feat/api-conventions
gh pr create --draft --title "feat!: API conventions (NewClient, pointer returns, Request naming)" \
  --body "Flat client kept; single-object returns now *T; params renamed to *Request; New→NewClient. Pre-1.0 breaking.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```
*(If starting fresh, create the worktree in Step 0 and do all edits there.)*

---

### Task 3: tinvest — REST facade receivers value→pointer

**Repo:** `tinvest`. **Only** change: REST service facade receivers value→pointer. Requests stay pointer, method names stay, gRPC untouched (parity constraint).

**Files:**
- Modify: `rest/client.go` (service fields → pointer types; wiring `xxx{cl}` → `&xxx{cl}`)
- Modify: `rest/instruments.go`, `rest/marketdata.go`, `rest/operations.go`, `rest/orders.go`, `rest/sandbox.go`, `rest/signals.go`, `rest/stoporders.go`, `rest/users.go` (receiver `(s xxxServiceClient)` → `(s *xxxServiceClient)`)

**Interfaces — produced:** identical public method set and signatures; only the (unexported) receiver kind changes, so no caller-visible break except the struct-field types.

- [ ] **Step 1: Flip service field types (rest/client.go)**

Change each `Instruments instrumentsServiceClient` → `Instruments *instrumentsServiceClient` (and the other 5: `operationsServiceClient`, `ordersServiceClient`, `sandboxServiceClient`, `signalsServiceClient`, `usersServiceClient`; plus any `marketDataServiceClient`/`stopOrdersServiceClient` present).

- [ ] **Step 2: Flip wiring (rest/client.go)**

In `NewClient`, change each `cl.Users = usersServiceClient{cl}` → `cl.Users = &usersServiceClient{cl}` for all services.

- [ ] **Step 3: Flip receivers in each service file**

In every `rest/<service>.go`, change all `func (s xxxServiceClient) Method(...)` → `func (s *xxxServiceClient) Method(...)`. Run per file:
Run: `grep -rn "func (s [a-z]*ServiceClient)" rest/`
Expected after edits: no non-pointer receivers remain (all show `func (s *...`).

- [ ] **Step 4: Build + test (incl. parity tests)**

Run: `go build ./... && go test ./...`
Expected: PASS, including `rest/paths_test.go` and `grpc/parity_test.go` (method set unchanged, so parity holds).

- [ ] **Step 5: Lint**

Run: `task lint 2>/dev/null || golangci-lint run`

- [ ] **Step 6: Worktree + commit + PR**

```bash
git worktree add -b feat/api-conventions .claude/worktrees/api-conventions origin/main
git add -A && git commit -m "refactor: use pointer receivers on REST service facades"
git push -u origin feat/api-conventions
gh pr create --draft --title "refactor: pointer receivers on REST service facades" \
  --body "Aligns REST facades with the shared convention (pointer receiver/fields). Method names & pointer requests unchanged (gRPC parity). gRPC untouched.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

### Task 4: alor — introduce facades + pointer returns + Request naming

**Repo:** `alor` (largest). Flat methods → 6 service facades; value returns → `*T`/`[]T`; `XxxParams` → `<Service><Method>Request`; `ServerTime` stays flat.

**Files:**
- Modify: `client.go` (add 6 `*xxxService` fields + wiring in `NewClient`; keep `ServerTime` if defined here or in wrappers)
- Create: `orders.go`, `stoporders.go`, `ordergroups.go`, `portfolio.go`, `trades.go`, `marketdata.go` (one file per service; move methods here with new names/receivers)
- Delete/empty: `wrappers.go`, `trading.go` (methods relocated; keep `ServerTime` on `*Client` — move it to `client.go`)
- Modify: request struct definitions (currently in `wrappers.go`/`trading.go`) — relocate alongside their service, renamed to `<Service><Method>Request`
- Modify: `client_test.go`, `facade_test.go`, `decode_test.go`, `scalars_test.go` (update call sites to `client.<Service>.<Method>` and new request/return types)

**Interfaces — produced (public surface):** per the taxonomy in the spec. Service facade pattern:
```go
// orders.go
type ordersService struct{ c *Client }

// OrdersListRequest ... (renamed from GetOrdersParams; fields unchanged)
type OrdersListRequest struct { /* ...existing fields... */ }

func (s *ordersService) List(ctx context.Context, req OrdersListRequest) (*ResponseOrdersHeavy, error) {
	// body identical to old GetOrders, but:
	//   - receiver s.c instead of c
	//   - return do[*ResponseOrdersHeavy](ctx, s.c, ...)  // pointer T
	return do[*ResponseOrdersHeavy](ctx, s.c, http.MethodGet, clientPath(...), q, nil, hooks...)
}
```

**Full method map (old `*Client` method → new `Service.Method`), request-type renames follow `<Service><Method>Request`:**

| Service | New method | Old method | Request type (was `*Params`) |
|---|---|---|---|
| Orders | List | GetOrders | OrdersListRequest |
| Orders | Get | GetOrder | OrdersGetRequest |
| Orders | PlaceMarket | PlaceMarketOrder | OrdersPlaceMarketRequest |
| Orders | PlaceLimit | PlaceLimitOrder | OrdersPlaceLimitRequest |
| Orders | ReplaceMarket | ReplaceMarketOrder | OrdersReplaceMarketRequest |
| Orders | ReplaceLimit | ReplaceLimitOrder | OrdersReplaceLimitRequest |
| Orders | Cancel | CancelOrder | OrdersCancelRequest |
| Orders | CancelAll | CancelAllOrders | OrdersCancelAllRequest |
| Orders | Estimate | EstimateOrder | OrdersEstimateRequest |
| Orders | EstimateBatch | EstimateOrders | OrdersEstimateBatchRequest |
| StopOrders | List | GetStopOrders | StopOrdersListRequest |
| StopOrders | Get | GetStopOrder | StopOrdersGetRequest |
| StopOrders | Place | PlaceStopOrder | StopOrdersPlaceRequest |
| StopOrders | PlaceLimit | PlaceStopLimitOrder | StopOrdersPlaceLimitRequest |
| StopOrders | Replace | ReplaceStopOrder | StopOrdersReplaceRequest |
| StopOrders | ReplaceLimit | ReplaceStopLimitOrder | StopOrdersReplaceLimitRequest |
| OrderGroups | List | ListOrderGroups | OrderGroupsListRequest |
| OrderGroups | Get | GetOrderGroup | OrderGroupsGetRequest |
| OrderGroups | Create | CreateOrderGroup | OrderGroupsCreateRequest |
| OrderGroups | Update | UpdateOrderGroup | OrderGroupsUpdateRequest |
| OrderGroups | Delete | DeleteOrderGroup | OrderGroupsDeleteRequest |
| Portfolio | Summary | GetSummary | PortfolioSummaryRequest |
| Portfolio | Positions | GetPositions | PortfolioPositionsRequest |
| Portfolio | Position | GetPosition | PortfolioPositionRequest |
| Portfolio | PositionsByLogin | GetPositionsByLogin | PortfolioPositionsByLoginRequest |
| Portfolio | Risk | GetRisk | PortfolioRiskRequest |
| Portfolio | FortsRisk | GetFortsRisk | PortfolioFortsRiskRequest |
| Portfolio | RiskRates | GetRiskRates | PortfolioRiskRatesRequest |
| Trades | Get | GetTrades | TradesGetRequest |
| Trades | Symbol | GetSymbolTrades | TradesSymbolRequest |
| Trades | History | GetTradeHistory | TradesHistoryRequest |
| Trades | SymbolHistory | GetSymbolTradeHistory | TradesSymbolHistoryRequest |
| Trades | All | GetAllTrades | TradesAllRequest |
| Trades | AllHistory | GetAllTradesHistory | TradesAllHistoryRequest |
| MarketData | Search | SearchSecurities | MarketDataSearchRequest |
| MarketData | SecuritiesByExchange | GetSecuritiesByExchange | MarketDataSecuritiesByExchangeRequest |
| MarketData | Security | GetSecurity | MarketDataSecurityRequest |
| MarketData | Boards | GetAvailableBoards | MarketDataBoardsRequest |
| MarketData | FuturesQuote | GetActualFuturesQuote | MarketDataFuturesQuoteRequest |
| MarketData | Quotes | GetQuotes | MarketDataQuotesRequest |
| MarketData | CurrencyPairs | GetCurrencyPairs | MarketDataCurrencyPairsRequest |
| MarketData | OrderBook | GetOrderBook | MarketDataOrderBookRequest |
| MarketData | History | GetHistory | MarketDataHistoryRequest |
| (flat `*Client`) | ServerTime | ServerTime | (no request) |

- [ ] **Step 1: Add facade fields + wiring (client.go)**

In `type Client struct`, add:
```go
	Orders       *ordersService
	StopOrders   *stopOrdersService
	OrderGroups  *orderGroupsService
	Portfolio    *portfolioService
	Trades       *tradesService
	MarketData   *marketDataService
```
At the end of `NewClient`, before `return`:
```go
	c.Orders = &ordersService{c}
	c.StopOrders = &stopOrdersService{c}
	c.OrderGroups = &orderGroupsService{c}
	c.Portfolio = &portfolioService{c}
	c.Trades = &tradesService{c}
	c.MarketData = &marketDataService{c}
```

- [ ] **Step 2: Create the 6 service files**

For each service, create `<service>.go` with `package alor`, the `type xxxService struct{ c *Client }`, and move every method from the map into it: new name, pointer receiver `(s *xxxService)`, body identical except `c` → `s.c` and the `do[T]`/`exec` return type changed to `*T` (single) — collections already return slices; keep those. Move each method's request struct next to it and rename per the table. Keep `ReqID`/`json:"-"` idempotency fields and hooks intact.

- [ ] **Step 3: Keep ServerTime flat (client.go)**

Move `func (c *Client) ServerTime(ctx context.Context) (int64, error)` into `client.go` unchanged (scalar return stays).

- [ ] **Step 4: Remove emptied files**

Delete `wrappers.go` and `trading.go` once all methods/structs are relocated. Confirm nothing else references them.

- [ ] **Step 5: Update tests**

In `client_test.go`, `facade_test.go`, `decode_test.go`, `scalars_test.go`: rewrite call sites to `client.Orders.List(ctx, OrdersListRequest{...})` etc., update request-type names, and change single-object result handling to pointers (`resp.Field` via `*T`; nil-check where needed). The `facade_test.go` likely enumerates the facade — update it to the new services.

- [ ] **Step 6: Build + test**

Run: `go build ./... && go test ./...`
Expected: PASS.

- [ ] **Step 7: Lint**

Run: `task lint 2>/dev/null || golangci-lint run`

- [ ] **Step 8: Worktree + commit + PR**

```bash
git worktree add -b feat/api-conventions .claude/worktrees/api-conventions origin/main
git add -A && git commit -m "feat!: group API into service facades, pointer returns, Request naming"
git push -u origin feat/api-conventions
gh pr create --draft --title "feat!: service facades + pointer returns + Request naming" \
  --body "Introduces Orders/StopOrders/OrderGroups/Portfolio/Trades/MarketData facades; value returns → *T; params → <Service><Method>Request; ServerTime stays flat. Pre-1.0 breaking. alormcp (version-pinned) unaffected.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

### Task 5: finam — pointer receivers + value requests + method renames

**Repo:** `finam`. Facades already exist. Change: receiver value→pointer; request pointer→value; rename facade types `xxxServiceClient`→`xxxService`; rename methods (drop prefix); rename request structs to new `<Service><Method>Request`.

**Files:**
- Modify: `client.go` (service field types `xxxServiceClient`→`*xxxService`; wiring `xxxServiceClient{c: client}`→`&xxxService{c: client}`)
- Modify: `accounts.go`, `assets.go`, `corporateactions.go`, `marketdata.go`, `orders.go`, `reports.go`, `sessions.go`, `usage.go` (facade type rename; receiver `(s xxxServiceClient)`→`(s *xxxService)`; method renames; request param `req *XxxRequest`→`req XxxRequest` (value); request struct renames)
- Modify: the matching `*_test.go` files (call sites, request-type names, value-request construction)

**Interfaces — produced (facade + renamed methods):**

| Service (field) | New method | Old method |
|---|---|---|
| Accounts | Get | GetAccount |
| Accounts | Trades | Trades |
| Accounts | Transactions | Transactions |
| Assets | List | Assets |
| Assets | All | AllAssets |
| Assets | Clock | Clock |
| Assets | Get | GetAsset |
| Assets | Constituents | GetConstituents |
| Assets | Params | GetAssetParams |
| Assets | Schedule | Schedule |
| Assets | OptionsChain | OptionsChain |
| Assets | Exchanges | Exchanges |
| CorporateActions | FutureBondsEvents | GetFutureBondsEvents |
| CorporateActions | PastBondsEvents | GetPastBondsEvents |
| CorporateActions | FutureDividends | GetFutureDividends |
| CorporateActions | PastDividends | GetPastDividends |
| CorporateActions | FutureSplits | GetFutureSplits |
| CorporateActions | PastSplits | GetPastSplits |
| MarketData | Bars | Bars |
| MarketData | OrderBook | OrderBook |
| MarketData | LastQuote | LastQuote |
| MarketData | LatestTrades | LatestTrades |
| Orders | List | GetOrders |
| Orders | Get | GetOrder |
| Orders | Place | PlaceOrder |
| Orders | Cancel | CancelOrder |
| Orders | PlaceSLTP | PlaceSLTPOrder |
| Reports | Create | CreateAccountReport |
| Reports | Info | GetAccountReportInfo |
| Sessions | TokenDetails | TokenDetails |
| Usage | Metrics | GetUsageMetrics |

Request struct renames follow `<Service><NewMethod>Request`, e.g. `AccountsGetAccountRequest`→`AccountsGetRequest`, `OrdersGetOrdersRequest`→`OrdersListRequest`, `OrdersPlaceOrderRequest`→`OrdersPlaceRequest`, `AssetsGetAssetParamsRequest`→`AssetsParamsRequest`, `CorporateActionsGetFutureBondsEventsRequest`→`CorporateActionsFutureBondsEventsRequest`, `ReportsCreateAccountReportRequest`→`ReportsCreateRequest`, `ReportsGetAccountReportInfoRequest`→`ReportsInfoRequest`. `AuthTokenDetailsRequest` is a generated body reused as the request — **keep its name** (generated); Sessions.TokenDetails passes it through.

- [ ] **Step 1: Rename facade types + fields + wiring (client.go + service files)**

Rename each `xxxServiceClient` → `xxxService` across all service files and `client.go`. In `client.go` change field types to pointer (`Accounts *accountsService`, …) and wiring to `client.Accounts = &accountsService{c: client}`.

- [ ] **Step 2: Flip receivers + method names + value requests (per service file)**

For each method: receiver `(s xxxServiceClient)` → `(s *xxxService)`; rename per the table; change `req *XxxRequest` → `req XxxRequest` and update the request struct name; inside the body, `req.Field` access is unchanged (value vs pointer is transparent for field reads); the `body` passed to `do` that was `req.Body` (a generated `*Body` pointer) stays a pointer field on the value struct — unchanged. For methods that passed `req` straight through as the body (`Sessions.TokenDetails`, `Reports.Create`), pass `&req`/`req` to match `do`'s `body any` (a value is fine).

- [ ] **Step 3: Rename request structs**

Rename each hand-written `type XxxRequest struct` to the new `<Service><NewMethod>Request` name (except the generated `AuthTokenDetailsRequest`). Keep all fields.

- [ ] **Step 4: Update tests**

In each `*_test.go`, update: facade type references, method names, request-type names, and construct requests as values (`OrdersPlaceRequest{...}` passed directly, not `&OrdersPlaceRequest{...}`).

- [ ] **Step 5: Build + test**

Run: `go build ./... && go test ./...`
Expected: PASS.

- [ ] **Step 6: Lint**

Run: `task lint 2>/dev/null || golangci-lint run`

- [ ] **Step 7: Worktree + commit + PR**

```bash
git worktree add -b feat/api-conventions .claude/worktrees/api-conventions origin/main
git add -A && git commit -m "feat!: pointer facade receivers, value requests, prefix-free method names"
git push -u origin feat/api-conventions
gh pr create --draft --title "feat!: pointer receivers, value requests, method renames" \
  --body "Aligns finam with the shared convention: pointer facade receivers/fields, value requests, methods drop redundant prefixes, request structs renamed <Service><Method>Request. Pre-1.0 breaking. Paired with finamcp update.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

### Task 6: finamcp — update to finam's new API (AFTER Task 5)

**Repo:** `finamcp` (consumes finam via `replace => ../finam`). Update all call sites; must build + test green against Task 5's finam.

**Files:**
- Modify: every `.go` file that calls `finam.<Method>` / constructs `finam.*Request` / accesses `client.<Service>` — locate with grep below.

- [ ] **Step 1: Find all finam call sites**

Run: `grep -rn "finam\." --include='*.go' .`
Expected: a list of constructor, facade-field, method, and request-type usages.

- [ ] **Step 2: Update call sites**

For each: `client.Accounts.GetAccount(ctx, &finam.AccountsGetAccountRequest{...})` → `client.Accounts.Get(ctx, finam.AccountsGetRequest{...})` (value request, new method name, renamed type). Apply the Task 5 method + request-type map. Facade field access is unchanged (still `client.Accounts`), but methods/types change.

- [ ] **Step 3: Build + test**

Run: `go build ./... && go test ./...`
Expected: PASS (against local `../finam` on branch `feat/api-conventions`).

- [ ] **Step 4: Lint**

Run: `task lint 2>/dev/null || golangci-lint run`

- [ ] **Step 5: Worktree + commit + PR**

```bash
git worktree add -b feat/api-conventions .claude/worktrees/api-conventions origin/main
git add -A && git commit -m "fix: update to finam service-facade API conventions"
git push -u origin feat/api-conventions
gh pr create --draft --title "fix: adopt finam API convention changes" \
  --body "Updates call sites for finam's new API (value requests, renamed methods/request types). Pairs with the finam conventions PR.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

## Self-review notes

- **Spec coverage:** every spec change-set item maps to a task — go-scaffolds (T1), moex (T2), tinvest (T3), alor (T4), finam (T5), finamcp (T6). Return rule, request naming, facade pattern, NewClient, method naming all appear in Global Constraints and per-task steps.
- **Type consistency:** request-type names are derived uniformly as `<Service><NewMethod>Request`; response types are left generated. `AuthTokenDetailsRequest` explicitly flagged as generated/unchanged.
- **Known judgment calls (finam):** `Assets.Assets`→`List` vs `AllAssets`→`All`; `Assets.GetAssetParams`→`Params`. If the implementer finds a clearer name that still follows the rule, note it in the PR for review.
