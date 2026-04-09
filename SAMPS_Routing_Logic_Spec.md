# SAMPS Routing Logic — Engineering Specification

> **Purpose:** This document defines the routing logic for the SAMPS Matrix tool, intended for implementation in Salesforce. It determines which SAMPS pod a seller should be routed to based on four inputs: Market, Industry, Category, and Products.
>
> **Live Prototype:** [https://ellieblock94.github.io/samps-matrix-v2/](https://ellieblock94.github.io/samps-matrix-v2/)

---

## 1. Input Fields

### 1.1 Market (Single Pick — Required)
| Value | Type |
|-------|------|
| US | Domestic |
| CA | Domestic |
| JP | International |
| AU | International |
| UK | International |
| EU | International |

### 1.2 Industry (Single Pick — Required)
| Value |
|-------|
| F&B |
| Retail |
| Health & Beauty |
| Professional Services |

### 1.3 Category
- **Single pick** when Market = US or CA
- **Multi pick** when Market = JP, AU, UK, or EU

| Value |
|-------|
| POS & Dashboard |
| Banking |
| Customers |
| eCommerce |
| Staff |
| Neighborhoods |

### 1.4 Products (Multi Pick)
Products are **dynamic** — the available options depend on the selected Category. "3P solutions" is always available regardless of Category.

| Category | Available Products |
|----------|-------------------|
| POS & Dashboard | F&B modes, Retail mode, Bookings mode, Invoices, Franchise setup, Kiosk, KDS, Drive-Thru, SRI by MarketMan |
| Banking | Checking, Savings, Loans, Credit Card, Bitcoin |
| Customers | Marketing, Loyalty, Messages, Customer Directory, Gift Cards |
| eCommerce | Websites, Online Ordering Profiles |
| Staff | Shifts, Payroll, Team Comms |
| Neighborhoods | Neighborhoods |
| *(All categories)* | 3P solutions |

When multiple categories are selected (international markets), products from all selected categories are combined.

---

## 2. Product Groups (Reference)

These product groups are referenced throughout the routing logic:

| Group Name | Products Included |
|------------|-------------------|
| **F&B Products** | F&B modes, Kiosk, KDS, Drive-Thru, SRI by MarketMan |
| **Services Products** | Bookings mode, Invoices |
| **Retail Products** | Retail mode |
| **Franchise Products** | Franchise setup |

---

## 3. Routing Logic

### 3.1 International Markets — Immediate Routing

These routes are determined **solely by Market selection**. Industry, Category, and Products do not affect the outcome, though the user should still complete all fields for data collection purposes.

```
IF Market = "JP"
    → ROUTE TO: JP SAMPS

IF Market = "AU"
    → ROUTE TO: AU SAMPS

IF Market = "UK" OR Market = "EU"
    → ROUTE TO: UK/EU SAMPS
```

### 3.2 Domestic Markets (US/CA) — Category-Based Routing

For non-POS categories, the route is determined **by Category alone**. Products do not affect the outcome.

```
IF Market IN ("US", "CA") THEN:

    IF Category = "Banking"
        → ROUTE TO: Banking SAMPS

    IF Category = "eCommerce"
        → ROUTE TO: eCom SAMPS

    IF Category = "Customers"
        → ROUTE TO: Customers SAMPS

    IF Category = "Staff"
        → ROUTE TO: Staff SAMPS

    IF Category = "Neighborhoods"
        → ROUTE TO: Neighborhoods SAMPS
```

### 3.3 Domestic Markets (US/CA) + POS & Dashboard — Priority-Based Routing

When Market is US or CA and Category is POS & Dashboard, the SAMPS pod is determined by a **strict priority evaluation**. Rules are evaluated top-to-bottom; the **first match wins**.

---

#### Priority #1: Franchise SAMPS ⚡ HIGHEST PRIORITY

**Franchise always wins. No exceptions. No overrides.**

```
IF Category = "POS & Dashboard"
   AND Products CONTAINS "Franchise setup"
THEN → ROUTE TO: Franchise SAMPS
```

- Overrides ALL other rules
- Industry does not matter
- Other selected products do not matter
- If Franchise setup is selected, the answer is always Franchise SAMPS

---

#### Priority #2: Services SAMPS

```
IF Industry IN ("Health & Beauty", "Professional Services")
   AND Category = "POS & Dashboard"
   AND Products CONTAINS ANY OF ("Bookings mode", "Invoices")
THEN → ROUTE TO: Services SAMPS
```

- Wins over Hybrid, F&B, and Retail
- Even if F&B products or Retail mode are also selected
- Only loses to Franchise (Priority #1)

---

#### Priority #3: Hybrid RTL/F&B SAMPS

Triggers on **any one** of these three conditions:

```
Condition A — Product Combination (any industry):
IF Category = "POS & Dashboard"
   AND Products CONTAINS "Retail mode"
   AND Products CONTAINS ANY OF ("F&B modes", "Kiosk", "KDS", "Drive-Thru", "SRI by MarketMan")
THEN → ROUTE TO: Hybrid RTL/F&B SAMPS

Condition B — Retail industry with F&B products:
IF Industry = "Retail"
   AND Category = "POS & Dashboard"
   AND Products CONTAINS ANY OF ("F&B modes", "Kiosk", "KDS", "Drive-Thru", "SRI by MarketMan")
THEN → ROUTE TO: Hybrid RTL/F&B SAMPS

Condition C — F&B industry with Retail product:
IF Industry = "F&B"
   AND Category = "POS & Dashboard"
   AND Products CONTAINS "Retail mode"
THEN → ROUTE TO: Hybrid RTL/F&B SAMPS
```

- Represents a cross-industry or cross-product mismatch
- Only loses to Franchise (#1) and Services (#2)

---

#### Priority #4: F&B SAMPS

```
IF Industry = "F&B"
   AND Category = "POS & Dashboard"
   AND Products CONTAINS ANY OF ("F&B modes", "Kiosk", "KDS", "Drive-Thru", "SRI by MarketMan")
   AND Products DOES NOT CONTAIN "Retail mode"
   AND Products DOES NOT CONTAIN "Franchise setup"
THEN → ROUTE TO: F&B SAMPS
```

---

#### Priority #5: Retail SAMPS

```
IF Industry = "Retail"
   AND Category = "POS & Dashboard"
   AND Products CONTAINS "Retail mode"
   AND Products DOES NOT CONTAIN ANY OF ("F&B modes", "Kiosk", "KDS", "Drive-Thru", "SRI by MarketMan")
   AND Products DOES NOT CONTAIN "Franchise setup"
THEN → ROUTE TO: Retail SAMPS
```

---

#### Priority #6: Fallback Rules (Industry-Agnostic)

When the selected industry does not align with the selected products, use these fallbacks:

```
IF Products CONTAINS ANY F&B Product
   AND Products DOES NOT CONTAIN "Retail mode"
THEN → ROUTE TO: F&B SAMPS

IF Products CONTAINS "Retail mode"
   AND Products DOES NOT CONTAIN ANY F&B Product
THEN → ROUTE TO: Retail SAMPS

IF Products CONTAINS ANY OF ("Bookings mode", "Invoices")
   AND Products DOES NOT CONTAIN ANY F&B Product
   AND Products DOES NOT CONTAIN "Retail mode"
THEN → ROUTE TO: Services SAMPS
```

---

## 4. Routing Priority Summary

| Priority | SAMPS Pod | Key Trigger |
|----------|-----------|-------------|
| 1 | **Franchise SAMPS** | Franchise setup selected (always wins) |
| 2 | **Services SAMPS** | H&B/Pro Services industry + Bookings/Invoices |
| 3 | **Hybrid RTL/F&B SAMPS** | Cross-industry mismatch or both Retail + F&B products |
| 4 | **F&B SAMPS** | F&B industry + F&B-only products |
| 5 | **Retail SAMPS** | Retail industry + Retail mode only |
| 6 | **Fallback** | Product-only matching when industry doesn't align |

---

## 5. All Possible SAMPS Outcomes

| SAMPS Pod | Possible Markets |
|-----------|-----------------|
| JP SAMPS | JP only |
| AU SAMPS | AU only |
| UK/EU SAMPS | UK, EU |
| Franchise SAMPS | US, CA |
| Services SAMPS | US, CA |
| Hybrid RTL/F&B SAMPS | US, CA |
| F&B SAMPS | US, CA |
| Retail SAMPS | US, CA |
| Banking SAMPS | US, CA |
| eCom SAMPS | US, CA |
| Customers SAMPS | US, CA |
| Staff SAMPS | US, CA |
| Neighborhoods SAMPS | US, CA |

---

## 6. UX Behavior Notes

These are behavioral notes from the prototype that may inform the Salesforce implementation:

1. **Progressive disclosure:** Sections enable as prior selections are made (Industry enables after Market, Category after Industry, Products after Category)
2. **Early result display:** The SAMPS result should display as soon as it can be determined:
   - International markets → immediately on market selection
   - Non-POS categories → immediately on category selection
   - POS & Dashboard → after product selection
3. **Dynamic product list:** Products shown depend on the selected category. For international markets with multi-pick categories, combine products from all selected categories.
4. **Category input type changes by market:** Single-pick for US/CA, multi-pick for international markets.
5. **3P solutions** is always available as a product option regardless of category. It does not affect routing.

---

## 7. Pseudocode (Complete)

```python
def route_to_samps(market, industry, category, products):
    """
    Returns the SAMPS pod name based on the four inputs.
    
    Args:
        market: str — one of: US, CA, JP, AU, UK, EU
        industry: str — one of: F&B, Retail, Health & Beauty, Professional Services
        category: str or list — single value for US/CA, list for international
        products: list of str — selected products
    
    Returns:
        str — SAMPS pod name
    """
    
    # ── International Markets (immediate routing) ──
    if market == "JP":
        return "JP SAMPS"
    
    if market == "AU":
        return "AU SAMPS"
    
    if market in ("UK", "EU"):
        return "UK/EU SAMPS"
    
    # ── Domestic Markets (US/CA) ──
    # Normalize category to string for US/CA
    cat = category if isinstance(category, str) else category[0]
    
    # Non-POS categories (immediate routing)
    CATEGORY_ROUTES = {
        "Banking": "Banking SAMPS",
        "eCommerce": "eCom SAMPS",
        "Customers": "Customers SAMPS",
        "Staff": "Staff SAMPS",
        "Neighborhoods": "Neighborhoods SAMPS",
    }
    
    if cat in CATEGORY_ROUTES:
        return CATEGORY_ROUTES[cat]
    
    # ── POS & Dashboard — Priority-based routing ──
    FB_PRODUCTS = {"F&B modes", "Kiosk", "KDS", "Drive-Thru", "SRI by MarketMan"}
    
    has_franchise = "Franchise setup" in products
    has_retail = "Retail mode" in products
    has_fb = bool(FB_PRODUCTS & set(products))
    has_bookings = "Bookings mode" in products
    has_invoices = "Invoices" in products
    has_services = has_bookings or has_invoices
    
    # Priority #1: Franchise (always wins)
    if has_franchise:
        return "Franchise SAMPS"
    
    # Priority #2: Services
    if industry in ("Health & Beauty", "Professional Services") and has_services:
        return "Services SAMPS"
    
    # Priority #3: Hybrid RTL/F&B
    if (has_retail and has_fb):                    # Condition A: both product types
        return "Hybrid RTL/F&B SAMPS"
    if (industry == "Retail" and has_fb):           # Condition B: Retail + F&B products
        return "Hybrid RTL/F&B SAMPS"
    if (industry == "F&B" and has_retail):          # Condition C: F&B + Retail mode
        return "Hybrid RTL/F&B SAMPS"
    
    # Priority #4: F&B
    if industry == "F&B" and has_fb and not has_retail:
        return "F&B SAMPS"
    
    # Priority #5: Retail
    if industry == "Retail" and has_retail and not has_fb:
        return "Retail SAMPS"
    
    # Priority #6: Fallback (industry-agnostic)
    if has_fb and not has_retail:
        return "F&B SAMPS"
    if has_retail and not has_fb:
        return "Retail SAMPS"
    if has_services and not has_fb and not has_retail:
        return "Services SAMPS"
    
    return "No matching SAMPS — review required"
```

---

## 8. Test Cases

| # | Market | Industry | Category | Products | Expected SAMPS |
|---|--------|----------|----------|----------|----------------|
| 1 | JP | F&B | POS & Dashboard | F&B modes | JP SAMPS |
| 2 | AU | Retail | Banking | Checking | AU SAMPS |
| 3 | UK | F&B | POS & Dashboard | F&B modes | UK/EU SAMPS |
| 4 | EU | Retail | Staff | Shifts | UK/EU SAMPS |
| 5 | US | F&B | Banking | Checking, Savings | Banking SAMPS |
| 6 | CA | Retail | eCommerce | Websites | eCom SAMPS |
| 7 | US | F&B | Customers | Marketing, Loyalty | Customers SAMPS |
| 8 | US | Retail | Staff | Shifts, Payroll | Staff SAMPS |
| 9 | US | F&B | Neighborhoods | Neighborhoods | Neighborhoods SAMPS |
| 10 | US | F&B | POS & Dashboard | Franchise setup | Franchise SAMPS |
| 11 | US | Health & Beauty | POS & Dashboard | Franchise setup, Bookings mode | Franchise SAMPS |
| 12 | US | Professional Services | POS & Dashboard | Bookings mode | Services SAMPS |
| 13 | US | Health & Beauty | POS & Dashboard | Invoices, F&B modes, Retail mode | Services SAMPS |
| 14 | US | Professional Services | POS & Dashboard | Bookings mode, F&B modes, Retail mode | Services SAMPS |
| 15 | US | F&B | POS & Dashboard | F&B modes, Retail mode | Hybrid RTL/F&B SAMPS |
| 16 | US | Retail | POS & Dashboard | Retail mode, Kiosk | Hybrid RTL/F&B SAMPS |
| 17 | CA | Retail | POS & Dashboard | F&B modes | Hybrid RTL/F&B SAMPS |
| 18 | US | F&B | POS & Dashboard | Retail mode | Hybrid RTL/F&B SAMPS |
| 19 | US | F&B | POS & Dashboard | F&B modes, Kiosk, KDS | F&B SAMPS |
| 20 | US | F&B | POS & Dashboard | Drive-Thru | F&B SAMPS |
| 21 | US | Retail | POS & Dashboard | Retail mode | Retail SAMPS |
| 22 | CA | Retail | POS & Dashboard | Retail mode, 3P solutions | Retail SAMPS |
| 23 | US | Retail | POS & Dashboard | Bookings mode | Services SAMPS (fallback) |
| 24 | US | F&B | POS & Dashboard | Retail mode, Franchise setup | Franchise SAMPS |
| 25 | US | Retail | POS & Dashboard | 3P solutions | No matching SAMPS |

---

## 9. Salesforce Implementation Notes

### Recommended Approach
- **Custom Object or Custom Metadata Type** to store the routing rules (allows updates without code deploys)
- **Flow or Apex Trigger** to evaluate the routing logic on case/opportunity creation or update
- **Picklist fields** on the relevant object for Market, Industry, Category
- **Multi-select picklist** for Products (or a related junction object for more flexibility)
- **Dependent picklists** to control which Products are available based on Category
- **Formula field or Flow variable** for the computed SAMPS Pod result

### Field-Level Dependencies
```
Market → controls Category input type (single vs. multi)
Category → controls available Product options
```

### Validation Rules
- Market is required
- Industry is required
- At least one Category must be selected
- At least one Product must be selected (for data completeness)

---

*Document generated from the live prototype. For the interactive version, visit the [SAMPS Matrix Navigator](https://ellieblock94.github.io/samps-matrix-v2/).*
