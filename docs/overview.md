# BotanIQal ERP – System Overview

This document explains what the BotanIQal ERP system does at a **conceptual level**.

BotanIQal is a supplement brand built on freeze-dried microgreens (and other powders).  
The ERP system connects:

- Tray production of **fresh microgreens**
- Freeze dryer capacity and batch planning
- Raw material & packaging inventory
- Finished goods inventory (bottled products)
- Batch-level financials and feasibility

---

## 1. Main Goals

1. **Plan monthly production** based on:
   - Product demand (units per SKU)
   - Fresh microgreen yields
   - Freeze dryer capacity and machine hours

2. **Check feasibility** before committing:
   - Enough raw materials on hand?
   - Enough machine time in the month?

3. **Track inventory accurately**:
   - Planned resource usage (from production plan)
   - Actual resource usage and packaging variance
   - Cycle counts submitted via Google Form

4. **Maintain batch traceability**:
   - Each monthly batch has a **Batch ID**
   - Each batch is logged into a master sheet
   - Each product’s production and expiration date are recorded

5. **Automate as much as possible**:
   - Pull yield data from a centralized Harvest Yields Summary
   - Push resource usage and bottling data to dedicated inventory workbooks
   - Auto-apply cycle counts directly to the Resource Ledger
   - Send tray production alerts via email

---

## 2. Major Files & Their Roles

### 2.1 Production Log (Monthly)

- Acts as the **planning and execution hub** for a specific month.
- Calculates:
  - Trays needed, grams of powder per product, total freeze dryer hours.
  - Resource usage (powders, bottles, labels, capsules, etc.).
  - Feasibility: “Can we execute this plan given inventory and machine constraints?”

Outputs:

- Pushes a **Monthly Summary** to the `MASTER PRODUCTION SHEET`.
- Pushes planned **Resource Usage** to the BotanIQal Resource Ledger.
- Pushes **Production Actuals** (bottles produced) to the Product Inventory.

---

### 2.2 MASTER PRODUCTION SHEET

- Receives summarized rows from each monthly **Production Log**.
- Each row is a single product line for a batch, with shared batch metadata (Month, Batch ID, trays produced, hours, cost, revenue, etc.).
- Purpose:
  - Historical tracking of all batches.
  - Quick overview of which products were made in which batch.
  - Starting point for future analytics (profitability per batch, utilization, etc.).

---

### 2.3 Resource Ledger (BotanIQal)

- Represents **all raw materials & packaging** used by BotanIQal:
  - Powders, excipients, capsules
  - Bottles, labels, lids, seals, shrink bands, etc.

Key mechanisms:

- **Planned usage**:
  - From the monthly Production Log’s `Resource Usage` sheet.
  - Pushed and subtracted from inventory via `pushResourceUsageToInventory()`.

- **Actual variance**:
  - From the `Production Actuals` sheet.
  - If packaging usage differs from plan, `applyPerProductPackagingVariance()` adjusts inventory.

- **Cycle counts**:
  - Submitted via a Google Form into `Cycle Count (Responses)`.
  - `autoApplyCycleCount(e)` updates On Hand and Last Count Date.

The result is a **reconciled view** of inventory that combines:
- Planning data,
- Actual usage, and
- Physical counts.

---

### 2.4 Product Inventory

- Tracks **finished goods** (bottled products).
- Key columns:
  - Batch ID
  - Product Name
  - Bottles Produced
  - Date Produced
  - Expiration Date

Data source:

- Updated via `pushProductionActualsToProductInventory()` from each monthly Production Log.

This lets you:

- See how many bottles exist per batch.
- Know when each batch expires.
- Perform simple FIFO / FEFO logic externally if needed.

---

### 2.5 Harvests Yield Summary (Global)

- A separate spreadsheet that summarizes **fresh tray yields** from actual harvests.
- Each row links:
  - Microgreen variety
  - Average yield (oz) per tray

Used by:

- `updateFreshYieldsFromMaster()` in the Production Log to populate the **Input Guide**.
- This ensures that freeze-dryer loads and powder planning are based on actual recent tray performance.

---

### 2.6 Tray Production Schedule

- A calendar-like view of microgreen tray runs that eventually feed the freeze dryer.
- `sendBotanIQProductionAlerts()` emails daily:
  - Soaks, drains, sowing, lights, and harvests.

This bridges the gap between **fresh production (trays)** and **value-added production (freeze-dried supplements)**.

---

## 3. Philosophy

This ERP system follows a few principles:

1. **Use Google Sheets as the “database”** when starting small  
   It’s easy to share, audit, and manually override when needed.

2. **Use Apps Script to glue everything together**  
   Instead of manually copy-pasting, scripts move data between:
   - Production Logs
   - Master batch logs
   - Inventory ledgers
   - Cycle count forms
   - Product batch logs

3. **Separate “Planning” vs “Actuals”**  
   The system distinguishes:
   - Planned resource usage (`Resource Usage`)
   - Actual packaging variance (`Production Actuals`)
   - Physical counts (`Cycle Count (Responses)`)

4. **Keep batch traceability at the center**  
   Everything revolves around consistent **Batch IDs** pushed into master tables.

---

## 4. Where This Could Go

In the future, this design could be:

- Ported into a relational database (PostgreSQL, MySQL).
- Fronted by a custom app (e.g., React + Supabase / Firebase).
- Integrated with more formal QC logs, COA tracking, and regulatory documentation.

For now, it’s a **working, production-grade system** running on spreadsheets and scripts.
