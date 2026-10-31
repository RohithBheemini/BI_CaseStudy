# 🛵 Zomato Last-Mile Delivery: Business Intelligence & SLA Optimization Case Study

An end-to-end data analytics and business intelligence solution analyzing **45,584 food delivery orders** across **22 Indian cities**, **440 restaurants**, and **1,320 delivery riders**. This project audits severe real-world data corruption, engineers operational KPIs, builds predictive ETAs, and delivers an executive-grade **Power BI Dashboard** with actionable operational levers to reduce 30-minute SLA breaches.

---

## 📊 Executive Summary & Key Metrics

In on-demand food delivery, **30 minutes** is the golden Service Level Agreement (**SLA**) standard:
- $\le \mathbf{30}\text{ min}$: On-time customer satisfaction and repeat retention.
- $> \mathbf{30}\text{ min}$: High delay risk, customer complaints, refund requests, and churn.

| Key Performance Indicator | Value | Business Significance |
| :--- | :---: | :--- |
| **Total Validated Orders** | **45,584** | Complete order volume analyzed across 22 metropolitan, urban & semi-urban markets |
| **On-Time SLA Compliance ($\le 30$ min)** | **70.16%** | $31,982$ orders successfully delivered within the 30-minute SLA window |
| **High Delay Risk / SLA Breach ($> 30$ min)** | **29.84%** | $13,602$ orders breached SLA — identifying the primary operational bottlenecks |
| **Average Actual Delivery Time** | **26.29 min** | Fleet-wide average from order placement to customer doorstep handoff |
| **Average Predicted Delivery (ETA)** | **26.25 min** | High-precision predictive baseline ($R^2 = 0.8113$, $\text{MAE} = 3.18\text{ min}$) |
| **Active Delivery Fleet** | **1,320 riders** | Average rider rating of **4.64 ★** across all active delivery personnel |
| **Average Delivery Distance** | **9.70 km** | Haversine distance between restaurant and customer doorstep |
| **Kitchen Pickup Wait Time** | **9.99 min** | Restaurant preparation & handoff delay before road transit begins |

---

## 🗂️ Project Repository Structure

```text
BI_CaseStudy/
│
├── Zomato_Business_Dashboard.pbip          # Main Power BI Desktop Project File
├── Zomato_Business_Dashboard.Report/       # Visuals, page definitions, layouts, and formatting
├── Zomato_Business_Dashboard.SemanticModel/# TMDL data model, DAX measures, and relationships
│
├── Zomato_Delivery_EDA_and_Cleaning.ipynb  # 7-stage Python data audit, cleaning & ML pipeline
├── zomato_delivery_dataset.csv             # Raw, uncleaned dataset (45,584 rows x 20 cols)
├── zomato_delivery_cleaned.csv             # Production dataset with engineered features (32 cols)
│
├── Zomato_Executive_Dashboard.html         # Interactive standalone HTML dashboard mockup
├── Zomato_Demo_Presentation.pptx           # Executive slide deck for live presentation
├── Zomato_Demo_Presentation.html           # Interactive web slide deck
├── Zomato_BI_Project_and_Pitch_Guide.md    # Comprehensive walkthrough & pitch script
├── README.md                               # Project documentation and reproduction guide
└── .gitignore                              # Git exclusions (Power BI cache, Python, OS temp)
```

---

## ⚙️ Data Engineering & Cleaning Pipeline

The raw data (`zomato_delivery_dataset.csv`) contains real-world data corruption that would distort business decisions if imported directly. The Jupyter Notebook [`Zomato_Delivery_EDA_and_Cleaning.ipynb`](file:///c:/Users/Rohith/Downloads/BI%20CaseStudy/Zomato_Delivery_EDA_and_Cleaning.ipynb) runs a context-aware 7-stage cleaning pipeline preserving **100% of the 45,584 orders**:

1. **String `'NaN'` Literals & Null Standardization**: Replaced textual `'NaN'` strings with true nulls across 8 corrupted columns and trimmed whitespace.
2. **Context-Aware Imputation**: Rather than dropping 9.3% of the dataset (`4,225` orders), missing cities were inferred via the `Delivery_person_ID` prefix (e.g., `DEHRES...` $\rightarrow$ Dehradun), and missing traffic was reconstructed using delivery hour bins.
3. **Geospatial Restoration (Null Island & Sign Flips)**:
   - Fixed negative coordinate flips (`Restaurant_latitude < 0`).
   - Repaired `4,071` records at Null Island `(0.0, 0.0)` by mapping restaurant IDs to known median GPS coordinates and calculating true Haversine travel distance.
4. **Timestamp Reconstruction & Kitchen Wait Extraction**:
   - Converted mixed-format timestamps (standard `HH:MM` strings vs. Excel decimal day fractions like `0.45833` $\rightarrow$ `11:00`).
   - Corrected midnight rollover (`Time_Orderd` > `Time_Order_picked`).
   - Separated total delivery into **Kitchen Pickup Delay** (prep wait) and **Road Transit Time**.
5. **Categorical Anomaly Remediation**:
   - Mapped out-of-spec `Vehicle_condition == 3` to standard condition `1` (Fair).
   - Filtered anomalous rider ages (`15`, `50`) and ratings (`6.0`).
6. **Feature Engineering (12 New Columns)**:
   - `Distance_km` (Haversine formula), `Pickup_Delay`, `Order_Time`, `Order_DayOfWeek`, `Order_Month`.
   - `Predicted_Time`, `Error`, `Absolute_Error`, and `Delay_Risk` (`Low`: $<22\text{m}$, `Medium`: $22\text{--}30\text{m}$, `High`: $>30\text{m}$).

---

## 📈 Power BI Dashboard Architecture

The Power BI project (`Zomato_Business_Dashboard.pbip`) is built on modern Power BI Project open-source metadata standards (TMDL + PBIR), structured into two executive-level analytical views:

### Page 1: Executive SLA & Business Outcomes (The 'What & Where')
- **Macro KPI Scorecard**: Total Orders (`45,584`), On-Time SLA Compliance (`70.16%`), High Risk SLA Breaches (`29.84%`), Average Delivery Time (`26.29m`), and Predicted ETA (`26.25m`).
- **Traffic Density Impact**: Proves that evening `Jam` traffic surges delivery time to **31.2 minutes** with a **50.6% SLA breach rate** (vs. **21.3 minutes** and 7.9% breach in `Low` traffic).
- **SLA Delay Risk Distribution**: Donut breakdown of `Low` ($15,749$ orders), `Medium` ($16,233$ orders), and `High Delay Risk` ($13,602$ orders).
- **Weather Severity Breakdown**: Demonstrates that `Fog` (**28.9m**) and `Cloudy` (**28.9m**) weather conditions add **+7 minutes** compared to `Sunny` weather (**21.9m**).
- **Urbanization City Tiers**: Compares `Urban` (**23.0m**), `Metropolitian` (**27.1m**), and `Semi-Urban` (**49.7m**, 100% breach due to long distance transit).

### Page 2: Fleet & Operational Drivers (The 'How to Fix It')
- **Fleet Scorecard**: Active Riders (`1,320`), Average Distance (`9.70 km`), Kitchen Pickup Delay (`9.99m`), and Average Rider Rating (`4.64 ★`).
- **Vehicle Type Efficiency**: Scooters and Electric Scooters deliver in **24.5 minutes** (averaging **3.1 minutes faster** than traditional motorcycles at **27.6 minutes**).
- **Multi-Order Batching Penalty**: Carrying 1 order averages **22.8 minutes** (safe), whereas batching 2 orders jumps to **40.5 minutes** and 3 orders to **47.8 minutes** (**100% SLA breach rate**).
- **Vehicle Maintenance Condition**: Poor Condition (`0`) vehicles add **+5.6 minutes** of delay (**30.1 minutes**) compared to well-maintained Fair/Good vehicles (**24.4 minutes**).
- **Order Category Analysis**: Shows uniform **9.99m** kitchen prep delay across `Meal`, `Snack`, `Buffet`, and `Drinks`.

---

## 💡 Top Strategic Recommendations

1. **Cap Multi-Order Batching at Peak Hours**: Restrict dispatch algorithms from assigning $\ge 2$ orders per rider during Jam traffic windows (19:00–22:30) to eliminate up to **32% of all SLA breaches**.
2. **Incentivize Electric Scooters & Scooters**: Promote two-wheeler scooter adoption in dense urban clusters to shave **3.1 minutes** per trip.
3. **Vehicle Maintenance Inspections**: Mandatory checkups for vehicles flagged with Condition `0`, directly recovering **5.6 minutes** in transit latency.
4. **Dynamic ETA Buffers for Adverse Weather**: Automatically expand customer-facing ETAs by **+7 minutes** during Fog/Cloud conditions to maintain customer trust and prevent delivery penalties.

---

## 🚀 Getting Started & Reproduction

### Prerequisites
- **Python**: Version 3.10+ with `pandas`, `numpy`, `matplotlib`, `seaborn`
- **Power BI Desktop**: July 2024 or later (supports `.pbip` Power BI Developer Projects)
- **Web Browser**: Any modern browser (Chrome, Edge, Firefox, Safari) for HTML dashboards & slides

### Running the Python Pipeline
```bash
# Clone the repository
git clone https://github.com/RohithBheemini/BI_CaseStudy.git
cd BI_CaseStudy

# Open and execute the Jupyter Notebook
jupyter notebook Zomato_Delivery_EDA_and_Cleaning.ipynb
```

### Opening the Power BI Dashboard
1. Open Power BI Desktop.
2. Select **File > Open report** and choose [`Zomato_Business_Dashboard.pbip`](file:///c:/Users/Rohith/Downloads/BI%20CaseStudy/Zomato_Business_Dashboard.pbip).
3. If prompted to refresh data, confirm connection to `zomato_delivery_cleaned.csv`.

### Viewing Presentations & Web Dashboard
- Open [`Zomato_Executive_Dashboard.html`](file:///c:/Users/Rohith/Downloads/BI%20CaseStudy/Zomato_Executive_Dashboard.html) directly in any web browser for an interactive zero-install dashboard experience.
- Open [`Zomato_Demo_Presentation.pptx`](file:///c:/Users/Rohith/Downloads/BI%20CaseStudy/Zomato_Demo_Presentation.pptx) or [`Zomato_Demo_Presentation.html`](file:///c:/Users/Rohith/Downloads/BI%20CaseStudy/Zomato_Demo_Presentation.html) for presentation slides.
- Refer to [`Zomato_BI_Project_and_Pitch_Guide.md`](file:///c:/Users/Rohith/Downloads/BI%20CaseStudy/Zomato_BI_Project_and_Pitch_Guide.md) for the live demo pitch script.

---

## 👤 Author
- **Rohith Bheemini**
- GitHub: [@RohithBheemini](https://github.com/RohithBheemini)
- Email: rohithbheemini25@gmail.com
