# SAMPS Matrix Navigator

Interactive routing tool that determines the correct SAMPS pod based on Market, Industry, Category, and Product selections.

## 🔗 Live Tool
**[Open SAMPS Matrix Navigator](https://ellieblock94.github.io/samps-matrix/)**

## How It Works
1. **Market** — Select a market (US, CA, JP, AU, UK, EU)
2. **Industry** — Select an industry vertical
3. **Category** — Select product category (multi-pick for international markets)
4. **Products** — Select specific products

The tool automatically routes to the correct SAMPS pod based on your selections.

## Routing Priority (US/CA + POS & Dashboard)
| Priority | SAMPS Pod | Trigger |
|----------|-----------|---------|
| 🥇 | Franchise SAMPS | Franchise setup selected (always wins) |
| 🥈 | Services SAMPS | H&B/Pro Services + Bookings/Invoices |
| 🥉 | Hybrid RTL/F&B SAMPS | Retail mode + F&B products, or cross-industry mismatch |
| 4 | F&B SAMPS | F&B industry + F&B-only products |
| 5 | Retail SAMPS | Retail industry + Retail mode only |
| 6 | Category-based | Banking, eCom, Customers, Staff, Neighborhoods |

## International Markets
- **JP** → JP SAMPS
- **AU** → AU SAMPS
- **UK/EU** → UK/EU SAMPS
