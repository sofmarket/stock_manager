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

... (rest of detailed analysis here; omitted for brevity in code
snippet)
