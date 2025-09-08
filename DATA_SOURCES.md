# Data Sources

This project uses publicly available secondary datasets. Links and brief descriptions are listed below. Always review each source’s terms of use/licensing before redistribution.

> **Last updated:** 8 Sep 2025 (Europe/London)

## Summary

| Source | Link | Contents (examples) | Role in Project |
|---|---|---|---|
| Hotel Booking Demand (Primary) | https://www.kaggle.com/datasets/jessemostipak/hotel-booking-demand | Booking status, lead time, market segment, deposit type, guest origin, dates | Core booking-level dataset for modeling cancellations |
| World Development Indicators (WDI) | https://datatopics.worldbank.org/world-development-indicators/ | Country-level macro indicators (e.g., GDP per capita, internet users, tourism metrics) | Macro context features (by country/year) |
| Hofstede Cultural Dimensions | https://geerthofstede.com/research-and-vsm/dimension-data-matrix/  •  https://www.kaggle.com/datasets/tarukofusuki/hofstedes-6d-model-of-national-culture | PDI, IDV, MAS, UAI, LTO, IVR by country | Cultural context features (by country) |
| UN M49 Standard Areas | https://unstats.un.org/unsd/methodology/m49/overview/ | ISO/M49 regions, subregions, country mappings | Region grouping & harmonisation |

---

## 1) Hotel Booking Demand (Primary Dataset)

- **Link:** https://www.kaggle.com/datasets/jessemostipak/hotel-booking-demand  
- **Notes:** Booking-level data for two Portuguese hotels including cancellation flag and rich attributes (lead time, deposit type, market segment, channel, special requests, guest origin).  
- **Usage:** Target variable (`is_canceled`), core features, and join key to guest-origin country.

## 2) World Development Indicators (WDI)

- **Link:** https://datatopics.worldbank.org/world-development-indicators/  
- **Notes:** Country/year macroeconomic & development indicators (e.g., GDP per capita, internet usage, tourism).  
- **Usage:** Joined to bookings via guest-origin country and booking year to provide macro context.

## 3) Hofstede Cultural Dimensions

- **Official matrix:** https://geerthofstede.com/research-and-vsm/dimension-data-matrix/  
- **Kaggle version:** https://www.kaggle.com/datasets/tarukofusuki/hofstedes-6d-model-of-national-culture  
- **Notes:** Six cultural dimensions: Power Distance (PDI), Individualism (IDV), Masculinity (MAS), Uncertainty Avoidance (UAI), Long-Term Orientation (LTO), Indulgence (IVR).  
- **Usage:** Joined by country to capture cultural context in cancellation behavior.

## 4) United Nations M49 (Regions & Areas)

- **Link:** https://unstats.un.org/unsd/methodology/m49/overview/  
- **Notes:** Standard UN country/area codes, regions, and subregions.  
- **Usage:** Region/subregion grouping during imputation.

---

## Licensing & Attribution

- Review and comply with each source’s **terms of use** and **attribution requirements** before sharing derived datasets.  
- If publishing, follow each provider’s recommended citation/attribution text.
