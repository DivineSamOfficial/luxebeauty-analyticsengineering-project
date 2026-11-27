# Use Case 1: Order & Sales Tracker
 
## From: Head of Commercial, LuxeBeauty
## To: Analytics Engineering
### Priority: High, needed before the festive-season planning review

Why we need this

Right now, every Monday, three teams send three different revenue numbers. Sales quotes order value, Finance quotes invoiced revenue, and Operations quotes what was delivered. Nobody can tell me why they differ. We also can’t see:

which channels and campaigns actually bring in customers who come back,
how long orders take to reach customers,
how much revenue we lose to returns and cancellations.

## 1. Business problem
Sales reports order value, Finance reports invoiced revenue, and Operations reports what was delivered, so there are three revenue numbers and nobody can explain the gap. The company also can't see channel or campaign effectiveness, fulfilment speed, or revenue lost to returns and cancellations. Goal: one trusted source, with every metric defined once.

## 2. Users
| User | Decision |
|---|---|
| Head of Commercial | Where to push: categories, regions, channels |
| Finance | Monthly revenue close; dashboard revenue must match the books |
| Operations | Slow fulfilment; warehouses falling behind |
| Marketing | Which channels and campaigns to fund |

## 3. Dashboards
**D1 Sales Performance:** KPI tiles (Booked Sales, Net Sales, Orders, AOV, Units, Gross Margin %, each vs. the previous period and the same period last year); Net Sales by month, this FY vs. last FY; actual vs. target by month and business unit (targets as a seed); sales by channel and B2B/B2C; by state; category → brand → product drill-down; top and bottom 10 products.

**D2 Order Fulfilment & Operations:** funnel in units (Ordered → Invoiced → Shipped → Delivered → Returned); open backlog ageing (0–2, 3–7, 8–15, 15+ days); lead times (order→dispatch, dispatch→delivery, order→delivery; median and P90); on-time delivery % (SLA 3 days B2C, 7 days B2B); fill rate; split-shipment rate, failed delivery attempts and RTO by warehouse and carrier; cancellation rate by channel and stage.

**D3 Customers & Marketing:** new vs. returning customers; monthly cohort retention grid (1–12 months); repeat purchase rate; average days between orders; revenue by marketing channel, first-touch vs. last-touch; campaign orders and revenue within campaign window; top customers; RFM segments (stretch).

**D4 Product, Pricing & Returns:** ABC/Pareto; discount depth (list vs. realised price) by category and channel; return rate by category, brand and product (units and value); return reasons and inspection outcome; festive season vs. normal months.

**Global filters:** date (calendar and FY April–March), business unit, category, brand, sales channel, customer type, state, region, warehouse.

## 4. Metric definitions
| Metric | Definition | Date |
|---|---|---|
| Booked Sales | Order line value after discount, excl. GST, non-cancelled orders | Order date |
| Net Sales (revenue) | Invoiced **product** value excl. GST minus credit notes; cancelled invoices excluded. Shipping-charge invoice lines are reported separately as Shipping Revenue | Invoice / credit note date |
| Orders | Distinct non-cancelled orders | Order date |
| AOV | Booked Sales ÷ Orders | Order date |
| Gross Margin | Net Sales − COGS (moving-average issue cost) | Invoice date |
| Fill Rate | Shipped qty ÷ ordered qty (non-cancelled lines) | Order date |
| On-Time Delivery % | Orders fully delivered within SLA ÷ orders delivered | Delivery date |
| Return Rate | Returned qty ÷ delivered qty (also by value) | Return date |
| New Customer | First-ever non-cancelled order falls in the period | Order date |
| Cancellation Rate | Cancelled orders ÷ all orders placed | Order date |

GST is never revenue.

## 5. Sources (schema `raw`) – final list, 30 tables
| Module | Tables |
|---|---|
| Sales (4) | orders, order_line_items, order_status_history, customer_payments |
| Invoicing (4) | invoices, invoice_line_items, credit_notes, credit_note_line_items |
| Shipping (4) | shipments, shipment_line_items, shipment_status_history, delivery_attempts |
| Returns (3) | sales_returns, sales_return_line_items, return_inspections |
| Inventory (1) | inventory_transactions (issue cost for COGS) |
| Master data (14) | companies, customers, customer_addresses, products, product_categories, product_brands, business_units, warehouses, states, shipping_carriers, sales_channels, marketing_channels, marketing_campaigns, employees |
| Seed | monthly sales targets (CSV) |

Added after reading the data dictionary: `companies` (financial-year start month drives dim_date), `employees` (field-sales rep on B2B orders). Deliberately excluded: shipment_tracking, refunds, customer_payment_allocations, accounts_receivable, inventory, inventory_reservations, payment_terms, payment_methods, all procurement/expense/accounting tables.

## 6. Rules
- Net Sales reconciles to invoices minus credit notes, to the rupee, as an automated test.
- Every fact has a documented grain.
- Keys tested unique and not_null; foreign keys tested with relationships.
- Daily refresh after the loader. Out of scope: procurement, inventory valuation, P&L.
- Master data sources have freshness disabled (they change rarely); transactional sources warn after 1 day, error after 3.

## 7. Acceptance criteria
1. Dashboards read only from `marts`.
2. `dbt build` passes with all tests green.
3. The Net Sales reconciliation test passes.
4. All mart models and columns documented.

## 8. Implied model design (outline)
| Fact | Grain | Serves |
|---|---|---|
| fct_order_lines | One product on one order | Booked Sales, AOV, units, fill rate, discounts |
| fct_invoice_lines | One invoice or credit note line (credit notes negative) | Net Sales, shipping revenue, GST, margin, reconciliation |
| fct_orders | One order with milestone dates (accumulating snapshot) | Lead times, SLA, funnel, backlog |
| fct_return_lines | One returned product on one return | Return rate, reasons |

Dimensions: dim_date (FY from companies, quarter, festive flags), dim_customers (segment, B2B/B2C, acquisition channel), dim_products (category tree flattened to 3 levels, brand + price tier + own-brand flag, business unit), dim_warehouses (DC vs store), dim_geography (state, region), dim_sales_channels, dim_marketing_channels, dim_campaigns, dim_carriers, dim_employees (sales reps).

Header/line rule: `order_line_items` drives fct_order_lines; header attributes from `orders` are joined down onto lines. Only measures at line grain are summed. Header-only amounts (e.g. `shipping_charge_amount`) must be allocated to lines or kept in fct_orders (fan-out risk).

### 8a. Findings from the data dictionary
| Question | Answer |
|---|---|
| Which warehouse fulfils SHIP orders? | `shipments` (one dispatch from one warehouse for one order). Counter sale / store pickup use `orders.store_warehouse_id` |
| How are cancelled invoices flagged? | Kept in `invoices`, marked cancelled (find the column) |
| Shipping charge | Appears as its own invoice line; separate it from product revenue |
| Credit note reasons | Returns, undelivered parcels (RTO) and price reductions, not only returns |
| Line-level progress | `order_line_items` carries counters: shipped, delivered, invoiced, cancelled, returned qty (current state; event tables give dates) |
| COGS | `inventory_transactions` has quantity and value per movement (sale shipped); link via shipment line → invoice line |
| Category tree | Up to 3 levels (self-referencing); categories also hold the return window days |
| First-touch attribution | `customers` holds how the customer first found the company |
| Address history | `customer_addresses` is versioned (new row + old marked deleted) |
| Financial year | `companies` holds the FY start month |
| Still open | Promised delivery date (use SLA days if absent); CLOSED vs DELIVERED (check `order_status_history` reasons) |

## 9. Build plan (per phase)
Sources → Base → Staging → Dimensions (dim_date first) → Facts → Tests and reconciliation → Dashboards as code.

## 10. Delivery phases (models built per phase; all sources declared up front)
| Phase | Modules | Delivers | Status |
|---|---|---|---|
| 1 (MVP) | sales + master_data | D1 on Booked Sales; D3 customers and channels; fct_order_lines, core dims | Sales sources done (PK tests pass; freshness stale as data ends 2026-11-17) |
| 2 | + invoicing | Net Sales, GST, revenue reconciliation, fct_invoice_lines | — |
| 3 | + shipping | D2 fulfilment, fct_orders milestones, warehouse for SHIP orders | — |
| 4 | + returns | D4 returns, fct_return_lines | — |
| 5 | + inventory | Gross Margin (COGS) | — |
 