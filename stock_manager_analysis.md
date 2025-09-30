# High-level architecture

-   **Frontend**: Vue 3 + Inertia (or Nuxt if you prefer SPA/SSR).
    TailwindCSS for UI.
-   **Backend**: Laravel 11 (API + web controllers via Inertia). Use
    Eloquent + DTOs.
-   **Database**: PostgreSQL (or MySQL). Prefer Postgres for advanced
    locking and analytic queries.
-   **Cache / Queue / Real-time**: Redis for cache + queues + pub/sub.
-   **Storage**: S3-compatible (MinIO or AWS S3) for
    invoices/attachments.
-   **Background workers**: Laravel Horizon / queues for heavy jobs
    (reports, notifications).
-   **Search/Analytics**: Optional --- ElasticSearch or ClickHouse for
    high-volume analytics.
-   **Infrastructure**: Docker + docker-compose for local, Kubernetes or
    managed containers for production.
-   **Monitoring / Logging**: Prometheus + Grafana + ELK (or Datadog).
-   **Authentication / Roles**: Laravel Breeze / Sanctum for SPA
    sessions; Role-based access control (RBAC).

------------------------------------------------------------------------

# Core domain model (conceptual / ER)

Entities (main): - `users` (with roles: admin, manager, salesperson) -
`clients` (name, mobile, email, address, notes) - `products` (sku, name,
description, unit, barcode) - `packs` (product bundles: pack =\> many
product items + quantity) - `purchase_invoices` (achats / shipping
invoices) - `purchase_lines` (lines for purchase invoices, link to
product, qty, unit_cost, tax) - `sale_invoices` (ventes / client
invoices) - `sale_lines` (product, qty, price, discount) - `stock_lots`
(incoming lots / batches) --- key for FIFO - `inventory_transactions`
--- ledger of stock movements (type: purchase, sale, adjustment, return,
transfer), with references to invoices and lot allocations -
`stock_reservations` --- reserved Qty for pending orders -
`notifications` --- e.g., min-stock alerts

ER relations: - `products` 1..\* `stock_lots` - `purchase_invoices`
1..\* `purchase_lines` - `sale_invoices` 1..\* `sale_lines` -
`stock_lots` create inventory via `purchase_lines` -
`inventory_transactions` reference `stock_lots` when qty is consumed
(sale)

------------------------------------------------------------------------

# Database schema (summary with main columns)

(Using Postgres types; adapt for MySQL)

## products

    id SERIAL PRIMARY KEY
    sku TEXT UNIQUE
    name TEXT
    description TEXT
    unit TEXT
    min_stock INT DEFAULT 0
    created_at, updated_at

## clients

    id SERIAL PRIMARY KEY
    name TEXT
    mobile TEXT
    email TEXT
    address TEXT
    notes TEXT
    created_at, updated_at

## purchase_invoices

    id SERIAL PRIMARY KEY
    invoice_no TEXT UNIQUE
    supplier_name TEXT
    received_at TIMESTAMP -- when goods received
    total_amount NUMERIC(12,2)
    created_by INT -> users.id
    created_at, updated_at

## purchase_lines

    id SERIAL PRIMARY KEY
    purchase_invoice_id INT -> purchase_invoices.id
    product_id INT -> products.id
    quantity INT
    unit_cost NUMERIC(12,4)
    tax NUMERIC(12,4)
    lot_reference TEXT -- optional from supplier
    created_at, updated_at

## stock_lots (core for FIFO)

    id SERIAL PRIMARY KEY
    product_id INT -> products.id
    origin_purchase_line_id INT -> purchase_lines.id NULLABLE
    total_quantity INT   -- original
    remaining_quantity INT -- current remaining
    unit_cost NUMERIC(12,4)
    received_at TIMESTAMP
    expiry_date DATE NULLABLE
    created_at, updated_at

## sale_invoices

    id SERIAL PRIMARY KEY
    invoice_no TEXT UNIQUE
    client_id INT -> clients.id
    total_amount NUMERIC(12,2)
    status ENUM ('draft','reserved','paid','cancelled')
    created_by INT -> users.id
    created_at, updated_at

## sale_lines

    id SERIAL PRIMARY KEY
    sale_invoice_id INT -> sale_invoices.id
    product_id INT -> products.id
    quantity INT
    unit_price NUMERIC(12,4)
    created_at, updated_at

## inventory_transactions

    id SERIAL PRIMARY KEY
    product_id INT
    stock_lot_id INT NULLABLE -- which lot was affected (NULL for adjustments)
    change INT -- positive for inbound, negative for outbound
    type TEXT -- 'purchase', 'sale', 'adjustment', 'return'
    reference_type TEXT -- 'purchase_line','sale_line',...
    reference_id INT
    cost NUMERIC(12,4) -- cost used when consuming inventory (for COGS)
    created_at TIMESTAMP

## stock_reservations

    id SERIAL PRIMARY KEY
    sale_invoice_id INT
    product_id INT
    quantity INT
    reserved_at TIMESTAMP
    expires_at TIMESTAMP NULLABLE
    created_at, updated_at

Indexes: index `stock_lots(product_id, received_at, remaining_quantity)`
for FIFO selection. Index
`inventory_transactions(product_id, created_at)` for reporting.

------------------------------------------------------------------------

# FIFO inventory approach (detailed)

**Principle**: When receiving goods (purchase), create a `stock_lot`
with `total_quantity` and `remaining_quantity`. When a sale occurs,
allocate outgoing quantity from the oldest lot(s) (smallest
`received_at`) with `remaining_quantity > 0`. Each allocation generates
an `inventory_transaction` row that records the change and cost used (so
you can compute COGS).

**Algorithm (transaction-safe)**

1.  Start DB transaction.
2.  For each `sale_line` (product, qty_to_consume):
    -   While qty_to_consume \> 0:
        -   Select the oldest available lot with remaining qty
            (`FOR UPDATE`).
        -   If none found → out of stock.
        -   Consume min(qty_to_consume, lot.remaining_quantity).
        -   Update `stock_lots.remaining_quantity`.
        -   Insert `inventory_transaction` (negative qty, cost).
        -   Decrease qty_to_consume.
3.  Commit transaction.

------------------------------------------------------------------------

# API endpoints

**Clients** - `GET /api/clients` - `POST /api/clients` -
`PUT /api/clients/{id}` - `DELETE /api/clients/{id}`

**Products & Packs** - CRUD endpoints - `POST /api/packs` -
`GET /api/products/{id}/barcode-print`

**Purchases** - `POST /api/purchases` - `GET /api/purchases/{id}`

**Sales** - `POST /api/sales` - `POST /api/sales/{id}/reserve` -
`POST /api/sales/{id}/finalize` - `GET /api/sales/{id}`

**Stock** - `GET /api/products/{id}/stock` - `POST /api/stock/adjust` -
`GET /api/stock/minimums`

**Reports** - `GET /api/reports/bestsellers` -
`GET /api/reports/clients-per-product` -
`GET /api/reports/client-history`

------------------------------------------------------------------------

# UI pages

-   Dashboard (stats, KPIs, min-stock alerts)
-   Clients (list + detail)
-   Products (list, detail with stock lots)
-   Packs (bundle products)
-   Purchases (create, receive goods)
-   Sales (draft → reserve → finalize)
-   Stock (lots, remaining qty, expiry)
-   Stats (bestsellers, clients per product, client history)
-   Notifications (min stock, system events)

------------------------------------------------------------------------

# Reporting queries

**Bestsellers**

``` sql
SELECT p.id, p.name, SUM(-it.change) AS qty_sold
FROM inventory_transactions it
JOIN products p ON p.id = it.product_id
WHERE it.type = 'sale'
GROUP BY p.id, p.name
ORDER BY qty_sold DESC;
```

**Clients per product**

``` sql
SELECT c.id, c.name, SUM(-it.change) AS qty_bought
FROM inventory_transactions it
JOIN sale_lines sl ON sl.id = it.reference_id AND it.reference_type = 'sale_line'
JOIN sale_invoices si ON si.id = sl.sale_invoice_id
JOIN clients c ON c.id = si.client_id
WHERE it.product_id = :product_id
GROUP BY c.id, c.name
ORDER BY qty_bought DESC;
```

------------------------------------------------------------------------

# Concurrency & correctness

-   Always consume lots inside DB transaction with row-level lock.
-   Use `stock_reservations` for availability before payment.
-   Decide policy for negative stock (better disallow).

------------------------------------------------------------------------

# Extensibility

-   Multi-warehouse: add `warehouse_id` to `stock_lots`.
-   Multi-currency: store currency in purchase/sale lines.
-   Internationalization: use Laravel localization.

------------------------------------------------------------------------

# Testing strategy

-   Unit tests for FIFO allocation (single/multi-lot, concurrent).
-   Integration tests for API.
-   Property-based tests for stock invariants.

------------------------------------------------------------------------

# Example Laravel code

``` php
DB::transaction(function() use ($productId, $qty) {
    while ($qty > 0) {
        $lot = StockLot::where('product_id', $productId)
            ->where('remaining_quantity', '>', 0)
            ->orderBy('received_at')
            ->lockForUpdate()
            ->first();

        if (!$lot) throw new OutOfStockException();

        $consume = min($qty, $lot->remaining_quantity);
        $lot->remaining_quantity -= $consume;
        $lot->save();

        InventoryTransaction::create([
            'product_id' => $productId,
            'stock_lot_id' => $lot->id,
            'change' => -$consume,
            'type' => 'sale',
            'cost' => $lot->unit_cost,
        ]);

        $qty -= $consume;
    }
});
```

------------------------------------------------------------------------

# Non-functional requirements

-   Security: RBAC, audit trail, backups.
-   Performance: indexes, caching, precomputed stock balances.
-   Monitoring: alerts on failed jobs, stock anomalies.
