# Kaveri Crunch ERP — Order-to-Audit Management System

Integrated Food Manufacturing ERP designed for a small-scale South Indian packaged snacks manufacturer (**Murukku**, **Mixture**, **Chips**, **Sweets**) in Tamil Nadu. It connects field sales ordering, order verification, prototype GST billing, dispatch, ledger-based inventory, BOM-based production, variance analysis, weighted-average product costing, and audit reporting in a single modular monolith.

---

## 1. Technology Stack & Justification

| Layer | Choice | Justification |
| :--- | :--- | :--- |
| Frontend | React 19, TypeScript, Vite, Tailwind CSS, Lucide, Recharts | Fast responsive UI across desktop dashboards and mobile-first salesman ordering |
| Offline & PWA | `vite-plugin-pwa`, IndexedDB (`idb`) | Field salesmen capture orders offline and sync idempotently via `clientOrderId` |
| Backend | Node.js 22, Express.js, TypeScript, Zod | Clean modular REST API with strict runtime input validation and backend RBAC |
| Database & ORM | PostgreSQL (Cloud SQL), Drizzle ORM | ACID transactions with `SELECT ... FOR UPDATE` row locking and type-safe SQL queries |
| Auth & Security | JWT (`jsonwebtoken` + `bcryptjs`), Firebase Auth, `express-rate-limit` | Dual support for 5-role ERP accounts and Google Sign-In, rate-limited auth endpoints |
| Exports | `exceljs`, `jspdf`, `jspdf-autotable` | Server-side generation of `.xlsx`, `.csv`, and `.pdf` invoices & audit reports |
| Testing | Vitest | Automated unit tests + full 9-step PostgreSQL integration benchmark |

> **ORM Note:** Cloud SQL in AI Studio uses Drizzle ORM with PostgreSQL (`drizzle-orm/node-postgres` + `drizzle-kit`) for schema management and transactional queries.

---

## 2. Demo User Accounts

All seeded role accounts share password **`Demo@1234`** (or sign in via Google Sign-In):

| Role | Email | Authorized Scope |
| :--- | :--- | :--- |
| **Owner** | `owner@kavericrunch.in` | Executive KPIs, charts, alerts, reports, benchmark controls |
| **Factory Admin** | `admin@kavericrunch.in` | Order verification, GST invoicing, dispatch, payments, purchases, master data |
| **Production Supervisor** | `supervisor@kavericrunch.in` | Raw/packaging stock, BOM issue, output capture, variance, overheads |
| **Salesman A (TN)** | `salesman1@kavericrunch.in` | Mobile order capture for assigned TN customers + IndexedDB offline sync |
| **Salesman B (KA)** | `salesman2@kavericrunch.in` | Mobile order capture for assigned Karnataka distributors |
| **Accountant / Auditor** | `auditor@kavericrunch.in` | Read-only access to all 17 reports, Excel/CSV/PDF exports, and audit log |

---

## 3. Core Workflows & Methods

### Inventory Ledger & Weighted-Average Valuation
Stock balances are **never directly overwritten**. Every receipt, production issue, production output, dispatch, return, and adjustment appends an immutable row to `inventory_movements` and updates `stock_positions` inside a PostgreSQL transaction using `SELECT ... FOR UPDATE`:

$$\text{Closing Stock} = \text{Opening} + \text{Receipts} - \text{Issues} - \text{Sales} + \text{Adjustments}$$

For outgoing issues, weighted-average cost is computed in integer paise using `BigInt` half-up rounding:

$$\text{Issue Value (paise)} = \text{roundHalfUp}\left(\frac{\text{Available Value (paise)} \times \text{Issued Qty}}{\text{Available Qty}}\right)$$

### BOM Production & Variance Tolerance (Default: 3%)
- **Consumption Variance %**: $\frac{\text{Actual} - \text{Planned}}{\text{Planned}} \times 100$
- **Yield Variance %**: $\frac{\text{Actual Good Packets} - \text{Expected Packets}}{\text{Expected Packets}} \times 100$
- Deviations exceeding $\pm 3\%$ (300 basis points) automatically set `hasVarianceAlert = true`.

### Product Costing
$$\text{Cost per Good Packet} = \frac{\text{Raw Material} + \text{Packaging (Good + Damaged)} + \text{Labour} + \text{Gas \& Power} + \text{Overheads}}{\text{Actual Good Packets Produced}}$$

---

## 4. Mandatory Benchmark Results (Verified via Automated Tests)

Running the 9-step benchmark (`npm run db:seed` or via the UI **Run 9-Step Benchmark** button) produces:

- **Flour Closing Stock**: `100 + 60 - 40 = 120 kg` (Weighted avg rate `₹43.125/kg`, consumed cost `₹1,725.00`)
- **Cooking Oil Closing Stock**: `30 - 10.5 = 19.5 litres` (Variance `+5.00%` flagged, cost `₹1,470.00`)
- **Spice Mix Closing Stock**: `5 - 1.2 = 3.8 kg` (Cost `₹240.00`)
- **200 g Pouches Closing Stock**: `500 - 188 - 6 = 306 pieces` (Cost `194 × ₹2.50 = ₹485.00`)
- **Cartons Closing Stock**: `30 - 9 = 21 pieces` (Cost `9 × ₹15.00 = ₹135.00`)
- **Murukku 200 g Finished Goods**: `50 + 188 - 150 = 88 packets`
- **Production Costing**: `₹1,725 + ₹1,470 + ₹240 + ₹485 + ₹135 + ₹1,200 + ₹600 = ₹5,855.00` (`₹31.14 / good packet`, yield variance `-6.00%` flagged)
- **Retailer A (TN) Invoice**: Taxable `₹7,500.00` + CGST `₹187.50` + SGST `₹187.50` = **`₹7,875.00`**; Outstanding after `₹4,000` payment = **`₹3,875.00`**
- **Distributor B (KA) Order**: `120` packets requested vs `88` available -> **`32 packets shortage`** flagged; approval/dispatch blocked. Inter-state Karnataka GST uses 5% IGST.

---

## 5. Local / Docker Setup & Commands

```bash
# Install dependencies
npm install

# Run automated unit & benchmark integration tests
npm test

# Reset and execute benchmark seed data
npm run db:seed

# Start full-stack server (port 3000)
npm run dev
```

Or via Docker Compose (`docker-compose.yml`) for local PostgreSQL + Node containerization.

---

## 6. Documented Assumptions & Known Limitations

- **Prototype GST Engine**: Uses the internship benchmark assumption of a flat 5% GST slab (2.5% CGST + 2.5% SGST intra-state; 5% IGST inter-state). Not certified for statutory GSTN portal filing, IRN/e-invoicing, or e-way bill generation.
- **Overhead Allocation**: Manufacturing overheads (labour, gas, power) are directly assigned to the production batch run and divided by actual good packets produced.
- **Single Factory Warehouse**: Stock locations are modeled for one factory (`MAIN-RM`, `MAIN-PKG`, `MAIN-FG`). Multi-branch stock transfers are out of scope.
