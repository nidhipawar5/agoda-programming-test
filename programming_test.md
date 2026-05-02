# Programming Framework Assessment – Submission

---

## Task 1: Evaluate and Revise the Pseudocode

### 1a. Evaluation

#### What is Good

1. **Data-driven config via `basicObj`**
   All fields and how to display them are defined in one place. To add a new field, you just add one entry to `basicObj`. The rendering loop itself does not need to change. This keeps config and logic separate, which is good.

2. **Named rendering options**
   Using named options like `"basic"` and `"object"` signals that different data types should be handled differently. It is a step toward a proper dispatch mechanism rather than hardcoding every field.

3. **Class abstraction for complex objects**
   Defining classes with `createString()` and `getSortVal()` shows that objects like `supplier` have their own internal structure. Keeping that logic inside the class is cleaner than scattering it throughout the loop.

---

#### What is Bad and Why

1. **`createString()` and `getSortVal()` are defined but never called**
   The pseudocode defines these methods, then ignores them — it manually writes `val.name` inline instead. This defeats the purpose of having a class at all. The class methods should be called during rendering.

2. **Supplier rendering is incomplete**
   Only `val.name` is rendered. Both `country` and `certifications` are in the input but are silently dropped. The output would be wrong even for the original requirements.

3. **The `"education"` class name is unexplained and unused**
   The `"Category"` entry passes `"education"` as a class name, but no such class exists and the loop has no branch that handles it. This creates confusion and a hidden bug.

4. **The `if/else if` chain does not scale**
   Every new rendering option requires adding another `else if` branch. A dispatch map (a dictionary that maps option names to handler functions) would make it easy to add new options without touching existing code.

5. **HTML tags are mixed directly into the logic**
   Writing `<b>` tags inline couples the output format (HTML) to the logic. Also, the pseudocode ends with `join by line break char` — but since `<b>` tags are already embedded, a plain `\n` newline will not render anything in HTML. You would need `<br>` instead. Separating content assembly from markup is cleaner and easier to maintain.

6. **No guard for missing or null values**
   The pseudocode assumes all fields are always present. If `certifications` is missing or `supplier` is null, the code would crash.

---

### 1b. Revised Pseudocode

```
/*
Assumptions:
  - Input is a valid JSON object as described in the spec.
  - Certifications may be an empty list or missing entirely.
  - `${variable}` means string interpolation.
  - sort() sorts a list alphabetically ascending by default.
*/


// ── CLASS DEFINITIONS ────────────────────────────────────────────────────────

CLASS Supplier:

    CONSTRUCTOR(data):
        self.name           = data['name']
        self.country        = data['country']
        self.certifications = data['certifications'] IF EXISTS ELSE []

    METHOD createString():
        // Returns a list of display lines, one per sub-field.

        sortedCerts = sort(self.certifications)   // ascending alphabetical

        IF sortedCerts is empty:
            certStr = "None"
        ELSE:
            certStr = join sortedCerts with ", "

        RETURN [
            `Supplier: ${self.name}`,
            `Country:  ${self.country}`,
            `Certifications: ${certStr}`
        ]

    METHOD getSortVal():
        RETURN self.name   // sort by supplier name when comparing instances


// ── RENDERING DISPATCH MAP ────────────────────────────────────────────────────
// Maps each rendering option to a handler function.
// To add a new option, add one entry here. No other code needs to change.

renderHandlers = {

    "basic": FUNCTION(key, val):
        RETURN [`${key}: ${val}`]

    "object": FUNCTION(key, val, className):
        // A second dispatch map for class names, so adding a new class
        // also requires only one new entry — no branching inside this handler.
        objectHandlers = {
            "supplier": FUNCTION(k, v): RETURN new Supplier(v).createString()
        }
        IF className EXISTS IN objectHandlers:
            RETURN objectHandlers[className](key, val)
        ELSE:
            LOG WARNING `No handler for object class '${className}'`
            RETURN []
}


// ── BASIC INFO OBJECT ─────────────────────────────────────────────────────────
// Each entry: key → [value, renderingOption, (optional) className]

basicObj = {
    "ID":       [ input['id'],          "basic"             ],
    "Name":     [ input['productName'], "basic"             ],
    "Category": [ input['category'],    "basic"             ],
    "Supplier": [ input['supplier'],    "object", "supplier"]
}


// ── RENDER BASIC INFO ─────────────────────────────────────────────────────────

basicTxtLst = []

FOR key, lst IN basicObj:

    val    = lst[0]
    option = lst[1]

    IF option NOT IN renderHandlers:
        LOG WARNING `Unknown rendering option '${option}' for key '${key}'`
        CONTINUE

    IF option == "object":
        className = lst[2]
        lines = renderHandlers[option](key, val, className)
    ELSE:
        lines = renderHandlers[option](key, val)

    ADD all items in lines TO basicTxtLst

OUTPUT = join basicTxtLst by "<br>" (HTML line break)
```

**Summary of improvements over the original:**

| Improvement | Reason |
|---|---|
| `Supplier.createString()` is actually called | Keeps formatting logic inside the class where it belongs |
| Nested `objectHandlers` dispatch map | Adding a new class is one dictionary entry — no branching inside the handler |
| `renderHandlers` dispatch map replaces `if/else if` | Adding a new rendering option is one dictionary entry, not a new branch |
| Country rendered as its own line | Satisfies the store managers' new request |
| Certifications sorted and joined inside the class | Satisfies alphabetical order request; empty/missing list is guarded |
| Warning logs for unknown options/classes | Fails visibly instead of silently dropping fields |
| `join by "<br>"` instead of plain newline | Consistent with `<b>` tags already in the output — plain `\n` would not render in HTML |

---

---

## Task 2: Stock and Sales Table – Pseudocode

**Choice: Stock and Sales Table**

---

### New Requirements

1. For each store: stock, last restocked date, units sold last month, units sold this month, and **sales trend (% change)**.
2. Show a **"Promotion Candidate"** banner if any store's sales trend is **≥ +20%** (retained from original spec).
3. Show a **"Promotion Alert"** banner if any store's sales trend **drops to −20% or below** (new requirement — a significant decline signals the product needs a promotional push).
4. A totals row at the bottom: **total stock**, **total units sold last month**, **total units sold this month**, **average sales trend** across all stores.

---

### Assumptions

- `salesTrend = ((unitsSoldThisMonth − unitsSoldLastMonth) / unitsSoldLastMonth) × 100`
- If `unitsSoldLastMonth == 0`, trend is undefined — display as `"N/A"` and exclude from the average calculation.
- Store names in `stockLevels` and `salesData` match exactly. The store name is the join key.
- A store may appear in `stockLevels` but have no matching entry in `salesData` (e.g. a brand-new store). Default sold values to `0` in that case.
- Thresholds are **inclusive** at the boundary: trend ≥ +20% triggers Promotion Candidate; trend ≤ −20% triggers Promotion Alert.
- The new requirement says "show a banner if sales trend drops below 20%" — this is ambiguous (below +20%, or a decline beyond −20%?). This pseudocode treats it as a significant decline (≤ −20%) because that is the scenario most likely to need a promotional response. This assumption should be confirmed with the store managers.
- The function returns both the rendered HTML table **and** a `bannerFlags` dictionary so the banner section can consume the computed flags without re-processing the data.

---

### Pseudocode

```
/*
FUNCTION: buildStockSalesTable(input)

INPUT  : product JSON (as specified in the assessment)
RETURNS: (tableHTML: string, bannerFlags: dict)
         tableHTML   → HTML string of the complete Stock and Sales Table
         bannerFlags → { "promotionCandidate": bool, "promotionAlert": bool }
*/

FUNCTION buildStockSalesTable(input):


// ── STEP 1: Build a lookup map from salesData by store name ──────────────────
//
//   Reason: avoids a nested loop (O(n²)) when matching sales records
//           to stock records. Lookup is O(1) per store.

    salesMap = {}                                  // storeName → salesRecord

    FOR each saleRecord IN input['salesData']:
        salesMap[ saleRecord['store'] ] = saleRecord


// ── STEP 2: Compute per-store row data ───────────────────────────────────────

    storeRows   = []    // list of row dicts, one per store
    trendValues = []    // numeric trend values for average (excludes N/A)

    FOR each stockRecord IN input['stockLevels']:

        storeName     = stockRecord['store']
        stock         = stockRecord['quantity']
        lastRestocked = stockRecord['lastRestocked']

        // --- Join sales data ---
        IF storeName EXISTS IN salesMap:
            soldLastMonth = salesMap[storeName]['unitsSoldLastMonth']
            soldThisMonth = salesMap[storeName]['unitsSoldThisMonth']
        ELSE:
            soldLastMonth = 0
            soldThisMonth = 0

        // --- Compute sales trend ---
        IF soldLastMonth == 0:
            trend    = null       // undefined; exclude from average
            trendStr = "N/A"
        ELSE:
            trend = ((soldThisMonth - soldLastMonth) / soldLastMonth) * 100

            // Prefix "+" for positive trends so the display is clear
            IF trend >= 0:
                trendStr = `+${round(trend, 1)}%`   // e.g. "+50.0%"
            ELSE:
                trendStr = `${round(trend, 1)}%`    // e.g. "-16.7%"

            ADD trend TO trendValues

        // --- Assemble row dict ---
        ADD {
            "store":          storeName,
            "stock":          stock,
            "lastRestocked":  lastRestocked,
            "soldLastMonth":  soldLastMonth,
            "soldThisMonth":  soldThisMonth,
            "trend":          trend,       // numeric or null (used for logic)
            "trendStr":       trendStr     // formatted string (used for display)
        } TO storeRows


// ── STEP 3: Compute totals / averages row ────────────────────────────────────

    totalStock    = SUM of row['stock']        FOR each row IN storeRows
    totalSoldLast = SUM of row['soldLastMonth'] FOR each row IN storeRows
    totalSoldThis = SUM of row['soldThisMonth'] FOR each row IN storeRows

    IF trendValues is NOT empty:
        avgTrend    = SUM(trendValues) / COUNT(trendValues)
        IF avgTrend >= 0:
            avgTrendStr = `+${round(avgTrend, 1)}%`
        ELSE:
            avgTrendStr = `${round(avgTrend, 1)}%`
    ELSE:
        avgTrendStr = "N/A"

    totalsRow = {
        "store":          "Total / Average",
        "stock":          totalStock,
        "lastRestocked":  "—",           // not applicable for an aggregate row
        "soldLastMonth":  totalSoldLast,
        "soldThisMonth":  totalSoldThis,
        "trend":          null,          // not used for banner logic on totals row
        "trendStr":       avgTrendStr
    }


// ── STEP 4: Determine banner flags ───────────────────────────────────────────
//
//   Computed here because the numeric trend values are already available.
//   The caller merges these flags with other banner sources (quality, restock).

    promotionCandidate = false
    promotionAlert     = false

    FOR each row IN storeRows:
        IF row['trend'] is NOT null:
            IF row['trend'] >= 20:
                promotionCandidate = true
            IF row['trend'] <= -20:
                promotionAlert = true

    bannerFlags = {
        "promotionCandidate": promotionCandidate,
        "promotionAlert":     promotionAlert
    }


// ── STEP 5: Render HTML table ─────────────────────────────────────────────────

    // --- Helper: render a single <td> cell, with optional CSS highlight ---
    FUNCTION renderCell(value, cssClass=null):
        IF cssClass is NOT null:
            RETURN `<td class="${cssClass}">${value}</td>`
        ELSE:
            RETURN `<td>${value}</td>`

    // --- Helper: decide CSS class for a trend cell ---
    FUNCTION trendCssClass(trend):
        IF trend is null:     RETURN null
        IF trend >= 20:       RETURN "trend-up"     // green highlight
        IF trend <= -20:      RETURN "trend-down"   // red highlight
        RETURN null                                   // no highlight

    // --- Table header ---
    tableHeader = """
    <thead>
      <tr>
        <th>Store</th>
        <th>Stock</th>
        <th>Last Restocked</th>
        <th>Units Sold (Last Month)</th>
        <th>Units Sold (This Month)</th>
        <th>Sales Trend</th>
      </tr>
    </thead>
    """

    // --- Data rows (one per store) ---
    tableBody = "<tbody>"

    FOR each row IN storeRows:
        tableBody += "<tr>"
        tableBody += renderCell(row['store'])
        tableBody += renderCell(row['stock'])
        tableBody += renderCell(row['lastRestocked'])
        tableBody += renderCell(row['soldLastMonth'])
        tableBody += renderCell(row['soldThisMonth'])
        tableBody += renderCell(row['trendStr'], cssClass=trendCssClass(row['trend']))
        tableBody += "</tr>"

    // --- Totals row (visually distinct via CSS class) ---
    tableBody += `<tr class="totals-row">`
    tableBody += `<td><b>${totalsRow['store']}</b></td>`
    tableBody += renderCell(totalsRow['stock'])
    tableBody += renderCell(totalsRow['lastRestocked'])
    tableBody += renderCell(totalsRow['soldLastMonth'])
    tableBody += renderCell(totalsRow['soldThisMonth'])
    tableBody += renderCell(totalsRow['trendStr'])
    tableBody += "</tr>"

    tableBody += "</tbody>"

    tableHTML = `<table>${tableHeader}${tableBody}</table>`

    RETURN tableHTML, bannerFlags

END FUNCTION
```

---

### Design Decisions for Maintainability

| Decision | Reason |
|---|---|
| `salesMap` dictionary (Step 1) | O(1) lookup per store instead of a nested loop. Adding new sales fields requires only reading from the existing map entry — no loop restructure. |
| Separate `trend` (numeric) and `trendStr` (display string) per row | Logic (banner flags, highlighting) uses the numeric value; the table cell uses the pre-formatted string. Changing rounding from 1 decimal to 2 is a one-line change. |
| "+" prefix handled explicitly in the trend string | Makes it clear in the display that a positive trend is a rise. Consistent between per-store rows and the totals row. |
| `bannerFlags` dict returned from this function | The table module already has the numeric values, so it is the natural place to compute trend-based flags. The caller merges flags from multiple sources (quality review, restock, etc.) without either module depending on the other. |
| `renderCell()` helper (Step 5) | Centralises cell markup. Changing highlight style, adding a tooltip, or wrapping in a `<span>` requires one change in one place. |
| `trendCssClass()` helper (Step 5) | Keeps the visual styling threshold separate from the banner trigger threshold. If one changes independently of the other, only one function needs updating. |
| Totals row built as a separate dict mirroring `storeRows` structure | Allows the rendering loop to treat all rows uniformly. Adding a new column means updating the dict structure once and the renderer once. |

---

### Worked Example Using the Sample Input

Given the sample input:

| Store | Stock | Last Restocked | Sold Last Month | Sold This Month | Sales Trend |
|---|---|---|---|---|---|
| Downtown | 12 | 2024-04-10 | 30 | 25 | −16.7% |
| Airport | 2 | 2024-04-01 | 10 | 15 | +50.0% |
| **Total / Average** | **14** | — | **40** | **40** | **+16.7%** |

**Banner flags produced:**
- `promotionCandidate = true` (Airport trend = +50.0% ≥ +20%)
- `promotionAlert = false` (no store has trend ≤ −20%)

---

---

## Task 3: Testing Strategy

### Overview

The goal is to verify that the program: (1) correctly processes all valid inputs, (2) handles edge cases without crashing, and (3) does not silently produce wrong output. The approach uses **unit tests**, **integration tests**, **edge / boundary case tests**, **malformed input tests**, and **performance tests**, all tied together in a **regression suite** that runs automatically after every code change.

---

### 1. Unit Tests — Test Each Function in Isolation

Each function is tested on its own with known inputs and expected outputs.

| Function Under Test | Input | Expected Output |
|---|---|---|
| Sales trend formula | soldLast=10, soldThis=15 | +50.0% |
| Sales trend formula | soldLast=30, soldThis=25 | −16.7% |
| Sales trend formula | soldLast=0, soldThis=10 | null / "N/A" (no division by zero) |
| `Supplier.createString()` | certs=["ISO9001","EcoLabel"] | Certifications: "EcoLabel, ISO9001" (alphabetical) |
| `Supplier.createString()` | certs=[] | Certifications: "None" |
| `Supplier.createString()` | certs field missing | Certifications: "None" (missing key handled) |
| Banner flag: `promotionCandidate` | trend = +20.0% (boundary) | `true` |
| Banner flag: `promotionCandidate` | trend = +19.9% | `false` |
| Banner flag: `promotionAlert` | trend = −20.0% (boundary) | `true` |
| Banner flag: `promotionAlert` | trend = −19.9% | `false` |
| Average trend | trendValues = [+50.0, −16.7] | +16.7% |
| Average trend | trendValues = [] (all N/A) | "N/A" |
| `trendCssClass()` | trend = +25 | "trend-up" |
| `trendCssClass()` | trend = −25 | "trend-down" |
| `trendCssClass()` | trend = +10 | null (no class) |
| `trendStr` sign prefix | trend = +50.0 | "+50.0%" (with "+" prefix) |
| `trendStr` sign prefix | trend = −16.7 | "−16.7%" (no "+" prefix) |

---

### 2. Integration Test — Full Input to Full Output

Use the sample JSON from the assessment as a known baseline. Run the entire pipeline from JSON input to HTML output and check every output field:

- The rendered HTML table contains exactly 2 data rows + 1 totals row.
- Downtown row shows trend "−16.7%" with no highlight class.
- Airport row shows trend "+50.0%" with `class="trend-up"`.
- Totals row shows stock=14, soldLast=40, soldThis=40, avgTrend="+16.7%".
- `bannerFlags = { "promotionCandidate": true, "promotionAlert": false }`.
- Basic info section shows supplier name, country "Germany", certifications "EcoLabel, ISO9001" in that alphabetical order.
- "Restocking Needed" banner appears (Airport stock = 2 < 5).
- "Quality Review Required" banner appears (returns increased from 0 to 1).

This is the **golden test** — if this passes after a code change, the core happy path is intact.

---

### 3. Edge Case and Boundary Tests

| Scenario | What to Verify |
|---|---|
| A store appears in `stockLevels` but has no entry in `salesData` | Sold values default to 0; trend is "N/A"; no crash |
| All stores have `unitsSoldLastMonth = 0` | All trends are "N/A"; totals row avgTrend = "N/A"; no division by zero |
| `certifications` field is missing entirely from the input | Treated as empty list; displays "None"; no crash |
| Product has only one store | Totals row equals that one store's row; average trend equals that store's trend |
| Trend is exactly +20% (boundary) | `promotionCandidate = true` (inclusive threshold confirmed) |
| Trend is exactly −20% (boundary) | `promotionAlert = true` (inclusive threshold confirmed) |
| Very large numbers (e.g. stock = 999,999) | Numbers render correctly without overflow or formatting issues |
| `salesData` array is empty | All stores default to 0 sold; no crash |
| `qualityAssessment.returnsLastMonth` equals `returnsThisMonth` | No Quality Review banner triggered (returns did not increase) |

---

### 4. Malformed Input Tests

These test what happens when the input JSON is not well-formed or has the wrong data types.

| Scenario | What to Verify |
|---|---|
| `stockLevels` key is missing entirely | Function handles missing key gracefully; shows empty table or a clear error message; no crash |
| `quantity` is a string (e.g. `"12"`) instead of a number | Function handles type mismatch; either converts or logs a clear error |
| `lastRestocked` date is in the wrong format | Displays the raw value as-is; does not crash |
| `salesData` contains a store not found in `stockLevels` | Extra sales record is ignored; no crash |
| JSON input is completely empty (`{}`) | All fields default to safe values; no crash |

---

### 5. Performance / Volume Tests

The background states there are thousands of products. These tests check that the program stays fast at scale.

| Scenario | What to Verify |
|---|---|
| 10,000 products processed in sequence | Total processing time stays within an acceptable limit (e.g. under 10 seconds) |
| A product with 100 stores in `stockLevels` | Per-store rows all render correctly; no slowdown from the `salesMap` lookup |
| Running the full pipeline 1,000 times in a loop | Memory usage does not grow continuously (no memory leak) |

---

### 6. Regression Testing

After any change to the code, the **entire test suite** (all sections above) is re-run automatically. This is important when:

- A new banner type is added (must not break existing banner logic).
- The totals row calculation changes (must not break per-store row rendering).
- The rendering helper functions are modified (must not corrupt cell output).

Running all tests automatically after every change means that fixing one thing cannot silently break another.

---

### Summary of Testing Approach

```
Unit Tests
  └── Test each function in isolation with controlled inputs
  └── Cover normal cases, boundary values, and zero/null inputs
  └── Verify "+" prefix on positive trend strings

Integration Test (Golden Test)
  └── Run the full pipeline on the provided sample input
  └── Assert exact expected output for every output field

Edge Case Tests
  └── Missing fields, empty arrays, zero denominators, single-store products

Malformed Input Tests
  └── Missing keys, wrong data types, empty JSON object

Performance / Volume Tests
  └── Thousands of products, products with many stores, memory leak check

Regression Suite
  └── All of the above re-run automatically after every change
```

---

*End of submission.*
