
# Demo Creative Data Builder — Reusable Prompt Template
Use this prompt **as-is** with ChatGPT’s Conversation API. It generates fully synthetic demo data for **one customer per run**, plus a matching **contract PDF** and an **Excel sales workbook**.  
**Important:** Do **not** use cannabis products; product names must be generic and aligned to the chosen company segment.

---

## 🔧 Input Variables (surrounded with `{{ }}`)
Supply any/all. Omitted variables use the defaults below.

- `{{ CustomerName }}` – Name of the made‑up or existing customer. *(Default: generate a realistic name based on segment and style)*  
- `{{ CompanyType }}` – `"existing"` or `"fake"`. *(Default: "fake")*  
- `{{ CompanyURL }}` – Optional URL if existing. *(Default: empty)*  
- `{{ CompanySegment }}` – Business segment driving product types (e.g., **Burgers/Fast Food**, **Retail Apparel**, **Coffee Shop**, **Electronics**, **SaaS**, **Pet Supplies**, etc.). *(Default: "Burgers/Fast Food")*  
- `{{ Style }}` – Tone for branding: `"formal"` or `"fun"`. *(Default: "formal")*  
- `{{ ProductSource }}` – `"auto"` (infer from segment/style), `"bySegment"` (force segment‑based), or `"byList"`. *(Default: "auto")*  
- `{{ ProductList }}` – Optional comma‑separated list of explicit products when `ProductSource="byList"`. *(Default: empty)*  
- `{{ SalesDepth }}` – `"Simple"` (10–15 SKUs), `"Medium"` (20–60 SKUs), `"Heavy"` (100+ SKUs). *(Default: "Medium")*  
- `{{ TimeRange }}` – Natural language like `"one month"`, `"two months"`, `"90 days"`, `"Q1 2025"`, `"Aug–Oct 2024"`. *(Default: "90 days")*  
- `{{ Region }}` – State/country/market for compliance text. *(Default: "Michigan, USA")*  
- `{{ Currency }}` – ISO code—for pricing. *(Default: "USD")*  
- `{{ ContractFlavor }}` – `"Simple"`, `"Full"`, or `"Legalese"`. *(Default: "Full")*  
- `{{ DiscountSplitA }}` – Distributor funding share as percent (e.g., `25`). *(Default: 25)*  
- `{{ DiscountSplitB }}` – Retailer funding share as percent (e.g., `25`). *(Default: 25)*  
- `{{ BundleDeal }}` – Optional text like `"2 for 10"`, `"Buy 2 get 1"`. *(Default: empty)*  
- `{{ DailyUnitLimit }}` – Units per customer per day. *(Default: 4)*  
- `{{ GroupTabsBy }}` – Excel tab grouping; accept `"single"` (one sheet) or `"category"`, `"month"`, `"channel"`. *(Default: "single")*  
- `{{ RandomSeed }}` – Integer for deterministic generation. *(Default: generate one)*

---

## 📦 Output Artifacts
1. **Contract PDF** — Dreamfields‑style distributor–retailer agreement tailored to inputs (non‑cannabis), with math‑ready terms.  
2. **Excel Workbook** — Synthetic POS‑like sales data aligned to the products that appear in the contract.

> **Filenames:** Use kebab‑case with sanitized `{{ CustomerName }}` and include the channel if present, e.g.  
> `contract-{{ CustomerName | kebab }}.pdf` and `sales-{{ CustomerName | kebab }}.xlsx`.

---

## 🧮 Math Terms (must be computable)
Include these *as explicit calculations* in both the dataset and contract text:
- **SRP** (Suggested Retail Price) per SKU.
- **Promo Discount** = `SRP * ({{ DiscountSplitA }}% + {{ DiscountSplitB }}%)`.
- **Net Promo Price** = `SRP - Promo Discount`.
- **UnitsSold** per day with realistic variance across `{{ TimeRange }}`.
- **GrossRevenue** = `SRP * UnitsSold`.
- **PromoDeductionDistributor** = `SRP * UnitsSold * ({{ DiscountSplitA }}/100)`.
- **PromoDeductionRetailer** = `SRP * UnitsSold * ({{ DiscountSplitB }}/100)`.
- **NetRevenue** = `GrossRevenue - PromoDeductionDistributor - PromoDeductionRetailer`.
- Optional **Bundle math** if `{{ BundleDeal }}` is provided (explain clearly in the contract how the per‑unit effective price is derived).

---

## 🗂️ Excel Schema (minimum columns)
**Sheet Name:** 
- If `{{ GroupTabsBy }}="single"` → `Sales`  
- Else create one sheet per group with the same schema.

**Columns (all sheets):**
- `Date` (within `{{ TimeRange }}`)  
- `Channel` (`In-Store`, `Online`)  
- `SKU` (unique)  
- `ProductName`  
- `Category` (segment‑appropriate)  
- `UnitPrice_SRP ({{ Currency }})`  
- `UnitsSold`  
- `GrossRevenue ({{ Currency }})`  
- `PromoSplit_Distributor_pct` (={{ DiscountSplitA }})  
- `PromoSplit_Retailer_pct` (={{ DiscountSplitB }})  
- `PromoDeduction_Distributor ({{ Currency }})`  
- `PromoDeduction_Retailer ({{ Currency }})`  
- `NetRevenue ({{ Currency }})`  
- Optional: `BundleApplied (Y/N)`, `BundleDetails`, `StoreID`

**Row volume:**  
- `"Simple"`: 10–15 SKUs x daily rows across `{{ TimeRange }}`  
- `"Medium"`: 20–60 SKUs x daily rows  
- `"Heavy"`: 100+ SKUs x daily rows

---

## 📄 Contract Must‑Have Sections
Use the **{{ ContractFlavor }}** to tune tone and length. Default **Full** mirrors the prior Dreamfields template (genericized, no cannabis). Include:
1. **Parties & Dates** (use `{{ Region }}`, `{{ TimeRange }}` window).  
2. **Purpose** (distributor → retailer; demo data context OK).  
3. **Products Covered** — List **only a subset** of SKUs that appear in the Excel (top movers).  
4. **Promotional Terms** — Explicit math for SRP, discount split `{{ DiscountSplitA }}/{{ DiscountSplitB }}`, net promo price, bundle math if provided, and **daily unit limit {{ DailyUnitLimit }}**.  
5. **Payment Terms** — Net 30, interest for late balances (2%/month).  
6. **Delivery & Inventory** — Weekly PO cadence example.  
7. **Marketing & Display** — POS/asset obligations based on `{{ Style }}`.  
8. **Compliance & Warranties** — Generic retail compliance for `{{ Region }}` (no regulated‑substance language).  
9. **Termination** — 30‑day notice clause.  
10. **Signatures** — Distributor / Retailer blocks.  

> *Ensure figures in the contract correctly summarize data from the Excel (e.g., top 5 SKUs, total NetRevenue, average NetPromoPrice).*

---

## 🚦 Generation Rules
- **No cannabis** examples. Products must align to `{{ CompanySegment }}` or `{{ ProductList }}` (if supplied).  
- **Styling by `{{ Style }}`**:  
  - `"formal"` → neutral tone, subdued product names, clean contract prose.  
  - `"fun"` → playful product names, friendlier copy (still compliant).  
- If `{{ ProductSource }}="byList"`, **only** use `{{ ProductList }}` and map each to a reasonable `Category`.  
- Respect `{{ RandomSeed }}` for deterministic output when provided.  
- If `{{ CompanyType }}="existing"` and a `{{ CompanyURL }}` is provided, keep the brand voice compatible but **do not** browse/scrape; infer tone only.  
- When `{{ GroupTabsBy }}` is not `"single"`, ensure all tabs aggregate to the same contract totals.

---

## ✅ What to Return
- A **Contract PDF** and an **Excel workbook** generated from these rules.  
- A short confirmation note listing: the number of SKUs, date span interpreted from `{{ TimeRange }}`, and where the files were saved.

---

## 📋 The Prompt (copy/paste below)
**Send this exact block to ChatGPT (with your `{{ variables }}` filled):**

```
You are my Demo Creative Data Builder. Using the instructions in this message, generate synthetic demo data and outputs for ONE customer.

[Inputs]
CustomerName={{ CustomerName }}
CompanyType={{ CompanyType }}
CompanyURL={{ CompanyURL }}
CompanySegment={{ CompanySegment }}
Style={{ Style }}
ProductSource={{ ProductSource }}
ProductList={{ ProductList }}
SalesDepth={{ SalesDepth }}
TimeRange={{ TimeRange }}
Region={{ Region }}
Currency={{ Currency }}
ContractFlavor={{ ContractFlavor }}
DiscountSplitA={{ DiscountSplitA }}
DiscountSplitB={{ DiscountSplitB }}
BundleDeal={{ BundleDeal }}
DailyUnitLimit={{ DailyUnitLimit }}
GroupTabsBy={{ GroupTabsBy }}
RandomSeed={{ RandomSeed }}

[Task]
1) Create a realistic product catalog aligned to CompanySegment and Style (or use ProductList if provided). Do NOT use cannabis products. 
2) Generate a POS-like sales dataset across the TimeRange. Include columns and math defined in the Excel Schema. Ensure NetRevenue, Promo splits, and any Bundle math are correct.
3) Write a distributor–retailer contract in the selected ContractFlavor (default Full) using Region and Currency, listing a subset of the SKUs you generated. Include explicit math clauses (SRP, total discount funded by Distributor/Retailer per the splits, Net Promo Price, and the DailyUnitLimit). The contract totals must reconcile with the dataset.
4) Produce two files: (a) a Contract PDF, and (b) an Excel workbook. If GroupTabsBy ≠ "single", create separate sheets by that grouping and ensure grand totals match the contract.

[Output Requirements]
- Return download links for both files.
- Summarize: total SKUs, total UnitsSold, GrossRevenue, NetRevenue, and the interpreted date span.
- Use the RandomSeed for deterministic generation if present.
```

---

## ✨ Example Invocation (minimal)
```
CustomerName={{ CustomerName }}
CompanyType=fake
CompanyURL=
CompanySegment=Burgers/Fast Food
Style=fun
ProductSource=auto
ProductList=
SalesDepth=Simple
TimeRange=90 days
Region=Michigan, USA
Currency=USD
ContractFlavor=Full
DiscountSplitA=25
DiscountSplitB=25
BundleDeal=2 for 10
DailyUnitLimit=4
GroupTabsBy=single
RandomSeed=42
```
