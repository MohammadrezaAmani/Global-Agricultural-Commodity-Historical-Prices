<div align="center">

# 🌾 Global Agricultural Commodity Historical Prices
### Soft & Hard Commodities Time-Series for Climate Econometrics & Supply Chain Modeling

[![Kaggle](https://img.shields.io/badge/Platform-Kaggle-blue?logo=kaggle)]()
[![Data Format](https://img.shields.io/badge/Format-JSON-orange)]()
[![Files](https://img.shields.io/badge/Files-60+_Contracts-green)]()
[![Domain](https://img.shields.io/badge/Domain-Agri_Economics-9b59b6)]()
![Date](https://img.shields.io/badge/date-2026--07--04-blue)

*Historical price action for global grains, livestock, and soft commodities from major global exchanges (CME, ICE).*

</div>

## 📖 1. Executive Summary & Market Context
Agricultural commodities are fundamentally distinct from financial assets or energy markets; their pricing dynamics are heavily governed by non-financial, exogenous variables such as weather anomalies (El Niño/La Niña), soil moisture levels, geopolitical trade tariffs, and supply chain bottlenecks. 

This dataset provides a granular, daily OHLC (Open, High, Low, Close) historical record of over 60 agricultural instruments. Crucially, it does not only provide spot prices; it includes **specific futures contract months** (e.g., December corn vs. March corn). This distinction is vital for financial engineers, as it allows for the construction of continuous futures curves, term-structure analysis, and roll-yield calculations. It is an essential repository for researchers modeling food inflation, quantifying climate change impacts on crop yields, or developing statistical arbitrage algorithms for soft commodities.

---

## 🗂️ 2. Dataset Architecture & Exhaustive File Inventory

The [data/commodities/](./data/commodities/) directory is strictly categorized into three primary macro-sectors. Below is the detailed taxonomy of the assets included:

### A. Grains & Oilseeds (The Caloric Complex)
These are the foundation of global food supply, highly sensitive to weather and biofuel mandates.
*   **Corn (Maize):** `commodities-us-corn.json`, `commodities-us-corn-contracts.json`
*   **Soybean Complex:** `commodities-us-soybeans.json`, `commodities-us-soybeans-contracts.json`, `commodity_us_soybean_meal.json`, `commodities-us-soybean-meal.json`, `commodities-us-soybean-meal-contracts.json`, `commodity_us_soybean_oil.json`, `commodities-us-soybean-oil.json`, `commodities-us-soybean-oil-contracts.json`
*   **Wheat:** `commodities-us-wheat.json`, `commodities-us-wheat-contracts.json`, `commodities-london-wheat.json`, `commodity_london_wheat.json`, `commodities-spring-wheat.json`
*   **Others:** `commodities-oats.json`, `commodities-oats-contracts.json`, `commodities-rough-rice.json`, `commodities-rough-rice-contracts.json`, `commodities-rough-rice-p.json`, `commodity_rough_rice.json`

### B. Livestock & Dairy (Protein Complex)
Driven by feed costs (corn/soy) and herd dynamics, these assets exhibit strong mean-reverting tendencies.
*   **Cattle:** `commodity_live_cattle.json`, `commodities-live-cattle-p.json`, `commodities-feeder-cattle-cme.json`, `commodities-feeder-cattle-contracts.json`, `commodity_feed_cattle.json`
*   **Dairy:** `commodities-class-iii-milk-cme.json`, `commodities-class-iii-milk-futures-contracts.json`

### C. Soft Commodities (Tropical & Industrial Agri)
Highly geographic-specific, these are prone to disease (coffee rust), frost, and logistic constraints.
*   **Coffee:** `commodity_us_cocoa.json` *(Note: verify labeling internally)*, `commodity_london_coffee.json`, `commodities-london-coffee.json`, `commodities-us-coffee-c-contracts.json`
*   **Cocoa:** `commodities-us-cocoa.json`, `commodities-us-cocoa-contracts.json`, `commodity_london_cocoa.json`, `commodities-london-cocoa.json`
*   **Sugar:** `commodity_london_sugar.json`, `commodities-london-sugar.json`, `commodities-us-sugar.json`, `commodities-us-sugar-no11-contracts.json`, `commodities-sugar-16.json`
*   **Fiber & Wood:** `commodities-us-cotton.json`, `commodities-us-cotton-no.2-contracts.json`, `commodity_lumber.json`, `commodities-lumber.json`, `commodities-lumber-contracts.json`
*   **Citrus:** `commodity_orange_juice.json`, `commodities-orange-juice.json`, `commodities-orange-juice-contracts.json`

---

## 🧬 3. Deep Data Schema & Ingestion Logic

Each JSON file follows a standardized Datatables payload. 
**Raw Schema Example:**
```json
{
    "draw": null,
    "total": 4500,
    "filtered": 4500,
    "data": [
        ["450.25", "448.10", "452.80", "451.50", "<span class=\"high\" dir=\"ltr\">2.50</span>", "<span class=\"high\" dir=\"ltr\">0.55%</span>", "2023/10/15", "1402/07/23"]
    ]
}
```

⚠️ **Critical Data Quirks:**
1.  **Numeric Formatting:** Prices contain comma separators (e.g., `"1,250.50"`). Simple `float()` casting will throw errors.
2.  **HTML Encapsulation:** The `Change` and `Change%` fields contain `<span>` tags with directional classes (`class="low"` for bearish, `class="high"` for bullish). 
3.  **Spot vs. Contracts:** Files ending in `-contracts.json` represent specific expiry months. If building a continuous time-series, you must algorithmically stitch these together (e.g., using a "panama canal" or "roll on volume" method) to avoid artificial price jumps.

### 🛠️ Production-Ready Data Loader (Python)
```python
import json
import pandas as pd
import re

class AgriDataIngestor:
    @staticmethod
    def _clean_financial_series(series: pd.Series, is_percent: bool = False) -> pd.Series:
        """Strips HTML, removes commas/percent, casts to float."""
        cleaned = series.astype(str).apply(lambda x: re.sub(r'<[^>]+>', '', x))
        cleaned = cleaned.str.replace(',', '', regex=False)
        if is_percent:
            cleaned = cleaned.str.replace('%', '', regex=False)
        return pd.to_numeric(cleaned, errors='coerce')

    def load_agri_contract(self, filepath: str) -> pd.DataFrame:
        try:
            with open(filepath, 'r', encoding='utf-8') as f:
                raw = json.load(f)
            
            df = pd.DataFrame(raw['data'], columns=[
                'Open', 'Low', 'High', 'Close', 'Abs_Change', 'Pct_Change', 'Gregorian_Date', 'Jalali_Date'
            ])
            
            for col in ['Open', 'Low', 'High', 'Close', 'Abs_Change']:
                df[col] = self._clean_financial_series(df[col])
            df['Pct_Change'] = self._clean_financial_series(df['Pct_Change'], is_percent=True)
            df['Gregorian_Date'] = pd.to_datetime(df['Gregorian_Date'])
            
            return df.set_index('Gregorian_Date').sort_index()
        except Exception as e:
            print(f"Error parsing {filepath}: {e}")
            return pd.DataFrame()
```

---

## 🔬 4. Advanced Quantitative & Scientific Applications

This dataset is not just for plotting line charts; it is structured for rigorous academic and quantitative research:

### A. Term Structure & Roll Yield Analysis (Cost of Carry)
By comparing the spot price (`commodities-us-corn.json`) with a distant futures contract (`commodities-us-corn-contracts.json`), you can calculate the "Cost of Carry". If futures are higher than spot (Contango), the roll yield is negative. You can build a model to predict when agricultural markets shift from Contango to Backwardation based on inventory forecasts.

### B. Climate Econometrics (External Data Join)
Merge this dataset with geospatial weather data (e.g., NOAA drought monitors, ENSO index). You can build a ** Distributed Lag Nonlinear Model (DLNM)** to quantify exactly how a 1-degree temperature anomaly in the American Midwest propagates into corn price volatility over the subsequent 3 to 6 months.

### C. Seasonality Decomposition using STL
Agricultural markets exhibit extreme seasonal patterns (e.g., harvest lows, pre-planting highs). You can apply **STL (Seasonal and Trend decomposition using Loess)** to isolate the seasonal component, allowing you to trade the residual (non-seasonal) trend, effectively removing seasonal noise from your machine learning features.

---

## 📚 5. Rigorous Academic Citation
If this dataset contributes to your published research, quantitative model, or academic thesis, please cite it properly to support open-source data.

**BibTeX Format:**
```bibtex
@misc{amani_2024_agri,
  author       = {Mohammadreza Amani},
  title        = {Global Agricultural Commodity Historical Prices},
  year         = {2024},
  publisher    = {Kaggle},
  url          = {https://www.kaggle.com/datasets/mohammadrezaamani/global-agricultural-commodity-prices},
  note         = {Includes spot and futures contracts for grains, softs, and livestock.}
}
```
**APA Format:**
> Amani, M. (2024). *Global Agricultural Commodity Historical Prices* [Data set]. Kaggle. https://www.kaggle.com/datasets/mohammadrezaamani/global-agricultural-commodity-prices

**MLA Format:**
> Amani, Mohammadreza. "Global Agricultural Commodity Historical Prices." *Kaggle*, 2024, www.kaggle.com/datasets/mohammadrezaamani/global-agricultural-commodity-prices.

---

## 👤 6. Author & Contact
**Mohammadreza Amani**  
🔧 *Data Architect & Financial Engineer*  
🌐 GitHub: [MohammadrezaAmani](https://github.com/MohammadrezaAmani)  
✉️ Email: [more.amani@yahoo.com](mailto:more.amani@yahoo.com)

---
<div align="center">
<b>⚖️ Disclaimer:</b> <i>This data is provided strictly for educational, research, and algorithmic backtesting purposes. It does not constitute financial or trading advice.</i><br><br>
*If this high-resolution agricultural data saves you hundreds of hours of scraping, please consider giving it an ⬆️ <strong>Upvote</strong> on Kaggle!*
</div>