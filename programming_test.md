# Programming Framework Assessment – Submission

---

## Task 1: Evaluate and Revise the Pseudocode

### 1a. Evaluation

#### What is Good

1. **Data-driven configuration via `basicObj`**
   Mapping each field name (key) to a `[value, renderingOption, ...]` tuple centralises what to display and how to display it in one place. Adding a new field only requires adding a single entry to `basicObj`—the rendering loop does not need to change. This is good separation of configuration from logic.

2. **Rendering options pattern**
   The use of named options (`"basic"`, `"object"`) is a step toward a dispatch mechanism. It signals intent to treat different data types differently without hardcoding field-by-field logic throughout the loop.

3. **Class abstraction for complex objects**
   Defining classes with `createString()` and `getSortVal()` shows awareness that supplier (and similar objects) have their own internal structure that should be encapsulated, rather than scattered inline throughout the rendering loop.

---

#### What is Bad / Could Be Improved

1. **`createString()` and `getSortVal()` are defined but never called**
   The pseudocode defines these methods on the class, then ignores them—manually writing `val.name` inline instead. This defeats the purpose of having a class. The class methods should be invoked during rendering.

2. **Supplier rendering is incomplete**
   The only supplier field rendered is `val.name`. `country` and `certifications` are both part of the input spec but are silently dropped. This means the output would be wrong even for the original requirements.

3. **The `"education"` class name in the `basicObj` example is unexplained and unused**
   The `"Category"` entry passes `"education"` as a class name, but no such class is defined anywhere, and the rendering loop has no branch that handles it. This creates confusion and a latent bug.

4. **The `if/else if` chain does not scale**
   Every new rendering option or class name requires manually adding another `else if` branch. A dispatch map (dictionary of option → handler function) would make the code open for extension without modification.

5. **HTML tags are mixed directly into pseudocode logic**
   Embedding `<b>` tags inline couples the output format (HTML) to the logic layer. In pseudocode, separating *content assembly* from *markup formatting* is cleaner and easier to adapt to other output formats.

6. **No guard for missing or null values**
   The pseudocode assumes all fields are always present. If `certifications` is missing or `supplier` is null, the code would crash silently.

---

### 1b. Revised Pseudocode

```
/*
Assumptions:
  - Input is a valid JSON object as described in the spec.
  - Certifications may be an empty list.
  - `${variable}` denotes string interpolation.
  - sort() sorts a list alphabetically ascending by default.
*/


// ── CLASS DEFINITIONS ────────────────────────────────────────────────────────

CLASS Supplier:

    CONSTRUCTOR(data):
        self.name           = data['name']
        self.country        = data['country']
        self.certifications = data['certifications']   // list of strings

    METHOD createString():
        // Returns a list of display lines, one per sub-field.

        sortedCerts = sort(self.certifications)        // ascending alphabetical

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
// To add a new option, add one entry here – no other code changes needed.

renderHandlers = {

    "basic": FUNCTION(key, val):
        RETURN [`${key}: ${val}`]

    "object": FUNCTION(key, val, className):
        IF className == "supplier":
            obj = new Supplier(val)
            RETURN obj.createString()
        ELSE:
            LOG WARNING `No handler for object class '${className}'`
            RETURN []
}


// ── BASIC INFO OBJECT ─────────────────────────────────────────────────────────

// Each entry: key → [value, renderingOption, (optional) className]
basicObj = {
    "ID":       [ input['id'],          "basic"            ],
    "Name":     [ input['productName'], "basic"            ],
    "Category": [ input['category'],    "basic"            ],
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

OUTPUT = join basicTxtLst by line break character
```

**Key improvements over the original:**

| Improvement | Reason |
|---|---|
| `Supplier.createString()` is actually called | Keeps formatting logic inside the class where it belongs |
| `renderHandlers` dispatch map | Replacing `if/else if` — adding a new option is one dictionary entry, not a new branch |
| Country rendered as its own line | Satisfies the store managers' new request |
| Certifications sorted and joined inside the class | Satisfies alphabetical order request; empty list guarded |
| Warning logs for unknown options/classes | Fails visibly instead of silently dropping fields |

---

---

## Task 2: Stock and Sales Table – Pseudocode

**Choice: Stock and Sales Table**

---

### New Requirements (as stated)

1. For each store: stock, last restocked date, units sold last month, units sold this month, **sales trend (% change)**.
2. Show a **"Promotion Candidate"** banner if any store's sales trend is ≥ +20% (retained from original).
3. Show a **"Promotion Alert"** banner if any store's sales trend **drops below −20%** (new — significant decline signals a product needs a promotional push).
4. Totals row at the bottom: **total stock**, **total units sold last month**, **total units sold this month**, **average sales trend** across all stores.

---

### Assumptions

- `salesTrend = ((unitsSoldThisMonth − unitsSoldLastMonth) / unitsSoldLastMonth) × 100`
- If `unitsSoldLastMonth == 0`, trend is undefined; display as `"N/A"` and exclude from the average.
- Store names in `stockLevels` and `salesData` match exactly — the store name is the join key.
- A store may appear in `stockLevels` but have no matching entry in `salesData` (e.g. new store). Default sold values to `0` in that case.
- Trend thresholds are **inclusive** at the boundary: trend ≥ +20% triggers Promotion Candidate; trend ≤ −20% triggers Promotion Alert.
- The function returns both the rendered HTML table **and** a `bannerFlags` dictionary so the caller (banner section) can consume the computed flags without re-processing the data.
- Output is rendered as an HTML `<table>`.

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

    salesMap = {}                                   // storeName → salesRecord

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
            trend    = ((soldThisMonth - soldLastMonth) / soldLastMonth) * 100
            trendStr = `${round(trend, 1)}%`    // e.g. "+50.0%" or "-16.7%"
            ADD trend TO trendValues

        // --- Assemble row dict ---
        ADD {
            "store":          storeName,
            "stock":          stock,
            "lastRestocked":  lastRestocked,
            "soldLastMonth":  soldLastMonth,
            "soldThisMonth":  soldThisMonth,
            "trend":          trend,       // numeric or null (for logic)
            "trendStr":       trendStr     // formatted string (for display)
        } TO storeRows


// ── STEP 3: Compute totals / averages row ────────────────────────────────────

    totalStock    = SUM of row['stock']        FOR each row IN storeRows
    totalSoldLast = SUM of row['soldLastMonth'] FOR each row IN storeRows
    totalSoldThis = SUM of row['soldThisMonth'] FOR each row IN storeRows

    IF trendValues is NOT empty:
        avgTrend    = SUM(trendValues) / COUNT(trendValues)
        avgTrendStr = `${round(avgTrend, 1)}%`
    ELSE:
        avgTrendStr = "N/A"

    totalsRow = {
        "store":          "Total / Average",
        "stock":          totalStock,
        "lastRestocked":  "—",            // not applicable for an aggregate row
        "soldLastMonth":  totalSoldLast,
        "soldThisMonth":  totalSoldThis,
        "trendStr":       avgTrendStr
    }


// ── STEP 4: Determine banner flags ───────────────────────────────────────────
//
//   Computed here (not in the banner module) because the numeric trend values
//   are already available; the caller merges these flags with other banner sources.

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

    // --- Helper: render a single <td> cell, optionally highlighted ---
    FUNCTION renderCell(value, cssClass=null):
        IF cssClass is NOT null:
            RETURN `<td class="${cssClass}">${value}</td>`
        ELSE:
            RETURN `<td>${value}</td>`

    // --- Helper: decide CSS class for a trend cell ---
    FUNCTION trendCssClass(trend):
        IF trend is null:          RETURN null
        IF trend >= 20:            RETURN "trend-up"      // green highlight
        IF trend <= -20:           RETURN "trend-down"    // red highlight
        RETURN null                                        // no highlight

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
    tableBody += `<td>${totalsRow['stock']}</td>`
    tableBody += `<td>${totalsRow['lastRestocked']}</td>`
    tableBody += `<td>${totalsRow['soldLastMonth']}</td>`
    tableBody += `<td>${totalsRow['soldThisMonth']}</td>`
    tableBody += `<td>${totalsRow['trendStr']}</td>`
    tableBody += "</tr>"

    tableBody += "</tbody>"

    tableHTML = `<table>${tableHeader}${tableBody}</table>`

    RETURN tableHTML, bannerFlags

END FUNCTION
```

---

### Design Decisions for Maintainability

| Decision | Rationale |
|---|---|
| `salesMap` dictionary (Step 1) | O(1) lookup per store instead of a nested loop. Adding new sales fields (e.g. `unitsSoldYearToDate`) requires only reading from the existing map entry — no loop restructure. |
| Separate `trend` (numeric) and `trendStr` (display string) per row | Keeps computation and formatting decoupled. Logic (banner flags, highlighting) uses the numeric value; the table cell uses the pre-formatted string. Changing rounding from 1 decimal to 2 is a one-line change. |
| `bannerFlags` dict returned from this function | The table module is the natural place to compute trend-based flags since it already has the numeric values. Returning a dict decouples this module from the banner renderer — the caller merges flags from multiple sources (quality review, restock, etc.) without either module depending on the other. |
| `renderCell()` helper (Step 5) | Centralises cell markup. Changing highlight style, adding a `title` tooltip, or wrapping in a `<span>` requires one change in one place. |
| `trendCssClass()` helper (Step 5) | Keeps the threshold logic for visual styling separate from the threshold logic for banner triggers. If the visual threshold changes independently of the banner threshold, only this function needs updating. |
| Totals row built as a separate dict mirroring `storeRows` structure | Allows the rendering loop to treat the totals row uniformly. Adding a new column means updating the dict structure once and the renderer once. |

---

### Worked Example Using the Sample Input

Given the sample input:

| Store | Stock | Last Restocked | Sold Last Month | Sold This Month | Sales Trend |
|---|---|---|---|---|---|
| Downtown | 12 | 2024-04-10 | 30 | 25 | −16.7% |
| Airport | 2 | 2024-04-01 | 10 | 15 | +50.0% |
| **Total / Average** | **14** | — | **40** | **40** | **+16.65%** |

**Banner flags produced:**
- `promotionCandidate = true` (Airport trend = +50.0% ≥ +20%)
- `promotionAlert = false` (no store has trend ≤ −20%)

---

---

## Task 3: Testing Strategy

### Overview

The goal is to verify that the program: (1) correctly processes all valid inputs, (2) handles edge cases without crashing, and (3) does not silently produce wrong output. I would use a combination of **unit tests**, **integration tests**, and **edge / boundary case tests**, followed by a **regression test suite** that runs automatically whenever the code changes.

---

### 1. Unit Tests — Isolated Functions

Each logical unit is tested independently with known inputs and expected outputs.

| Unit Under Test | Input | Expected Output |
|---|---|---|
| Sales trend formula | soldLast=10, soldThis=15 | +50.0% |
| Sales trend formula | soldLast=30, soldThis=25 | −16.7% |
| Sales trend formula | soldLast=0, soldThis=10 | null / "N/A" (no division by zero) |
| `Supplier.createString()` | certs=["ISO9001","EcoLabel"] | Certifications: "EcoLabel, ISO9001" (alphabetical) |
| `Supplier.createString()` | certs=[] | Certifications: "None" |
| Banner flag: `promotionCandidate` | trend = +20.0% (boundary) | `true` |
| Banner flag: `promotionCandidate` | trend = +19.9% | `false` |
| Banner flag: `promotionAlert` | trend = −20.0% (boundary) | `true` |
| Banner flag: `promotionAlert` | trend = −19.9% | `false` |
| Average trend | trendValues = [+50.0, −16.7] | +16.65% |
| Average trend | trendValues = [] (all N/A) | "N/A" |
| `trendCssClass()` | trend = +25 | "trend-up" |
| `trendCssClass()` | trend = −25 | "trend-down" |
| `trendCssClass()` | trend = +10 | null (no class) |

---

### 2. Integration Test — Full Input → Full Output

Use the sample JSON from the assessment as a known baseline. Run the entire pipeline from JSON input to HTML output and assert:

- The rendered HTML table contains exactly 2 data rows + 1 totals row.
- Downtown row shows trend "−16.7%", no highlight class.
- Airport row shows trend "+50.0%" with `class="trend-up"`.
- Totals row shows stock=14, soldLast=40, soldThis=40, avgTrend="+16.65%".
- `bannerFlags = { "promotionCandidate": true, "promotionAlert": false }`.
- Basic info section shows supplier name, country "Germany", certifications "EcoLabel, ISO9001" in that alphabetical order.
- "Restocking Needed" banner appears (Airport stock = 2 < 5).
- "Quality Review Required" banner appears (returns increased from 0 to 1).

This test serves as a **golden test** — if this single test passes after a code change, the core happy-path is intact.

---

### 3. Edge Case and Boundary Tests

| Scenario | What to Verify |
|---|---|
| A store appears in `stockLevels` but has no entry in `salesData` | Sold values default to 0; trend is "N/A"; no crash |
| All stores have `unitsSoldLastMonth = 0` | All trends are "N/A"; totals row avgTrend = "N/A"; no division by zero |
| `certifications` field is missing entirely from input | Code handles missing key gracefully (treats as empty list); displays "None" |
| Product has only one store | Totals row equals that one store's row; average trend equals that store's trend |
| Trend is exactly +20% (boundary) | `promotionCandidate = true` (inclusive threshold confirmed) |
| Trend is exactly −20% (boundary) | `promotionAlert = true` (inclusive threshold confirmed) |
| Very large numbers (e.g. stock = 999,999) | Numbers render without overflow or formatting issues |
| Product with no `salesData` array | Function handles empty array; all stores default to 0; no crash |
| `qualityAssessment.returnsLastMonth` equals `returnsThisMonth` | No Quality Review banner triggered (returns did not increase) |

---

### 4. Regression Testing

After any change to the codebase, the **entire test suite** (units + integration + edge cases) is re-run automatically. This is particularly important when:

- A new banner type is added (must not break existing banner logic).
- The totals row calculation changes (must not break per-store row rendering).
- The rendering helper functions are modified (must not corrupt cell output).

Running tests automatically on every code change ensures that fixing one thing does not silently break another.

---

### Summary of Testing Approach

```
Unit Tests
  └── Test each function in isolation with controlled inputs
  └── Cover normal cases, boundary values, and zero/null inputs

Integration Test (Golden Test)
  └── Run the full pipeline on the provided sample input
  └── Assert exact expected output for every output field

Edge Case Tests
  └── Missing fields, empty arrays, zero denominators, single-store products

Regression Suite
  └── All of the above re-run automatically after every change
```

---

*End of submission.*
