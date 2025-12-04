
# Contract Analysis with Owed Amounts — **Merged & Robust Prompt**

This template **merges** your original “Contract Analysis with Owed Amounts” prompt with robust **very‑fuzzy matching**, **SRP derivation**, and **non‑zero math** handling.  
It preserves the **same outputs your system expects** (no structural changes), while eliminating the 0‑value failure cases caused by string mismatches or missing unit prices.

> **Note:** The Excel and PDF generation flows remain unchanged from your current pipeline. This prompt only strengthens the **matching + math** layer used by the contract analysis/owed amounts step.

---

## 🎯 Goal
Given a **contract JSON** (with `products[].name`) and **one or more POS Excel** files, compute owed amounts and a reconciled summary **without returning zeros** due to naming differences or missing unit prices.

- Use the **JSON** list as the **subset of SKUs** to include in the contract/owed amounts.
- Pull **Units / UnitPrice / GrossRevenue** from Excel.
- If a product in JSON doesn’t exactly match Excel, use **very fuzzy** matching (no hard threshold).
- If `UnitPrice` is missing in Excel but `GrossRevenue` & `UnitsSold` exist, **derive SRP** = `GrossRevenue / UnitsSold`.
- If both are missing, keep **N/A** (do **not** coerce to 0). Contract text may state “price unrestricted” when applicable.

---

## 🔧 Inputs
- `ContractJSON` — extracted contract object containing `products[].name` (the subset to include).
- `POSExcels[]` — one or more Excel files (e.g., MED + REC) with product names, quantities, and price and/or revenue columns.
- **Configurable Parameters (with defaults):**
  - `DiscountSplitA` (default **25**) — Distributor funding %
  - `DiscountSplitB` (default **25**) — Retailer funding %
  - `BundleDeal` (optional) — examples: `"2 for 10"`, `"Buy 2 get 1"`
  - `BundleOverride` (optional, default **false**) — if `true`, use the lower of **NetPromoPrice** vs **Bundle effective price**
  - `Currency` (default `"USD"`) — for display only

> The calling system’s **filenames and output locations** should remain as currently implemented; this prompt does not alter file naming.

---

## 🔎 Matching — **Very Fuzzy (No Hard Threshold)**
1. **Normalize** names (both JSON and Excel):
   - Lowercase
   - Strip punctuation
   - Collapse multiple spaces
   - Remove common pack/size tokens: `0.5g`, `1.3g`, `2g`, `x 3`, `x 5`, `3pk`, `5pk`, `pack`, etc.
2. **Score** each JSON product against all Excel products using:
   - `SequenceMatcher` similarity (**70% weight**),
   - Token **Jaccard** overlap (**30% weight**).
3. **Select the single best match** per JSON product (no hard threshold).  
   - **Log** `JSON_Product`, `Matched_Excel_Product`, and `MatchScore` for review.
4. If multiple Excel lines aggregate to the same `Matched_Excel_Product`, **aggregate** by product name prior to matching:
   - `UnitsSold = SUM(Units)`  
   - `GrossRevenue = SUM(Revenue)`  
   - `UnitPrice_SRP = MEAN(UnitPrice)` (if present; otherwise derived later)

---

## 💵 Pricing & Non‑Zero Handling
- If `UnitPrice_SRP` is **missing** but `GrossRevenue` & `UnitsSold` are present and `UnitsSold > 0`, set  
  **`SRP_Estimated = GrossRevenue / UnitsSold`**.
- If both are missing/unavailable, keep **`SRP_Estimated = N/A`**; do **not** force to zero.
- If `GrossRevenue` is missing but `SRP_Estimated` and `UnitsSold` exist, backfill:  
  **`GrossRevenue = SRP_Estimated * UnitsSold`**.

> This ensures calculations never collapse to 0 solely due to naming or missing price.

---

## 🧮 Owed Amounts & Contract Math (per SKU)
Use `DiscountSplitA` / `DiscountSplitB` (default 25/25).

- `PromoPerUnit = SRP_Estimated * (DiscountSplitA + DiscountSplitB)/100`
- `NetPromoPrice = SRP_Estimated - PromoPerUnit`
- `PromoDeduction_Distributor = UnitsSold * SRP_Estimated * (DiscountSplitA/100)`
- `PromoDeduction_Retailer = UnitsSold * SRP_Estimated * (DiscountSplitB/100)`
- `NetRevenue = GrossRevenue - PromoDeduction_Distributor - PromoDeduction_Retailer`

### Optional Bundle Math
- Support rules like:
  - **N-for-Price**: `"2 for 10"` → `EffectiveUnitPrice = 10 / 2`
  - **Buy-Get**: `"Buy 2 get 1"` → `EffectiveUnitPrice = (2 * SRP) / 3`
- If `BundleOverride = true`: set `NetPromoPrice = min(NetPromoPrice, EffectiveUnitPrice)` when both are available.
- Include `BundleApplied (Y/N)` and `BundleDetails` in the row output if a bundle is in effect.

---

## ✅ Validation & Diagnostics
- Output a **match log** with `JSON_Product`, `Matched_Excel_Product`, `MatchScore`.
- Output a **diagnostic list** for rows where:
  - `SRP_Estimated` is `N/A`,
  - `UnitsSold` is missing/`N/A`,
  - `GrossRevenue` is missing/`N/A`.
- Include **grand totals** and ensure they reconcile:
  - `TotalGross = Σ GrossRevenue`
  - `TotalDistributor = Σ PromoDeduction_Distributor`
  - `TotalRetailer = Σ PromoDeduction_Retailer`
  - `TotalNet = Σ NetRevenue`

---

## 📤 Outputs (unchanged from your system’s expectations)
Your existing pipeline already emits the Excel and PDF in the format your calling system expects. Keep those as‑is. This prompt guarantees the numbers are **non‑zero** and reconciled.

**Per‑SKU Output Columns (CSV/Excel detail):**
- `JSON_Product`
- `Matched_Excel_Product`
- `MatchScore`
- `UnitsSold`
- `SRP_Estimated`
- `GrossRevenue`
- `PromoSplit_Distributor_pct` (= `DiscountSplitA`)
- `PromoSplit_Retailer_pct` (= `DiscountSplitB`)
- `PromoDeduction_Distributor`
- `PromoDeduction_Retailer`
- `NetPromoPrice`
- `NetRevenue`
- Optional: `BundleApplied`, `BundleDetails`

**Summary PDF/Section (unchanged):**
- Totals: `Units`, `GrossRevenue`, `PromoDeduction_Distributor`, `PromoDeduction_Retailer`, `NetRevenue`
- Top N matched lines for quick review

---

## 🧠 Pseudocode (reference implementation)

```
1) Load ContractJSON → jsonProducts = [p.name]
2) Load POSExcels → concat all sheets into salesDF
3) Guess cols: nameCol, qtyCol, priceCol, revenueCol
4) Aggregate by nameCol:
   agg = SUM(qtyCol) as UnitsSold, SUM(revenueCol) as GrossRevenue, MEAN(priceCol) as SRP_raw
5) Normalize names → add _norm for agg and JSON
6) For each JSON product:
     i, score = bestMatch(jsonName_norm, agg._norm)
     matched = agg[i]
     SRP = SRP_raw
     if (SRP is NA) and (GrossRevenue, UnitsSold present and UnitsSold>0): SRP = GrossRevenue / UnitsSold
     if (GrossRevenue is NA) and (SRP, UnitsSold present): GrossRevenue = SRP * UnitsSold
     Compute PromoPerUnit, NetPromoPrice; bundle effective price if BundleOverride
     Compute PromoDeduction_Distributor, PromoDeduction_Retailer, NetRevenue
     Append to rows with JSON_Product, Matched_Excel_Product, score, all metrics
7) Emit detail CSV/Excel and summary PDF/section (formats unchanged)
```

---

## 📦 Configuration Defaults
- `DiscountSplitA = 25`
- `DiscountSplitB = 25`
- `BundleDeal = ""` (none)
- `BundleOverride = false`
- `Currency = "USD"`

---

## 🗣️ Contract Language Note (when SRP is N/A)
Include a clause like:
> “If no SRP is specified for a SKU, pricing is unrestricted, and promotional funding splits apply to the **actual realized price** at sale. Where available, SRP is derived as GrossRevenue ÷ UnitsSold solely for settlement math.”

This keeps the contract valid even when the data lacks explicit prices.
