# Zomato Last-Mile Delivery BI Case Study: Complete Project Understanding & Demo Pitch Guide

This guide breaks down the entire project **from scratch** in simple terms and provides a **step-by-step live demonstration script** for presenting both your Jupyter Notebook (`Zomato_Delivery_EDA_and_Cleaning.ipynb`) and your Power BI Dashboard (`Zomato_Business_Dashboard.pbip`).

---

## PART 1: Understanding the Project From Scratch (The "Why, What, and How")

### 1. What is the Business Problem?
In food delivery (like Zomato or Swiggy), **30 minutes** is the golden Service Level Agreement (**SLA**) benchmark.
- When an order takes **$\le 30$ minutes**, customer satisfaction and repeat order rates remain high.
- When an order takes **$> 30$ minutes** (**SLA Breach / High Delay Risk**), customer complaints spike, ratings drop, and Zomato incurs refund/coupon costs.
- **Our Goal**: Take Zomato's raw historical delivery logs (`45,584` orders across `22` Indian cities, `440` restaurants, and `1,320` riders), clean up severe real-world data corruption, engineer operational KPIs (distance, kitchen prep wait vs. road transit time, predicted ETA, and SLA risk tier), and build an interactive **Business Outcomes Dashboard** that tells operations managers **why delays happen** and **which levers they can pull to fix them**.

---

### 2. Understanding the Raw Dataset (`zomato_delivery_dataset.csv`)
The raw file has **45,584 rows** and **20 columns**, grouped into 5 logical categories:
1. **Identifiers (2 cols)**:
   - `ID`: Unique order ID (e.g., `0xcdcd`).
   - `Delivery_person_ID`: Structured code `<CITY>RES<RESTAURANT_NO>DEL<RIDER_NO>` (e.g., `DEHRES17DEL01` = Dehradun, Restaurant #17, Delivery Rider #01). **Crucial insight**: Even when the `City` column or GPS coordinates are missing/corrupted, this ID tells us the exact city and restaurant!
2. **Rider Profile (2 cols)**:
   - `Delivery_person_Age`: Age of the rider (should be 20–39).
   - `Delivery_person_Ratings`: Customer rating of the rider (should be 1.0–5.0 stars).
3. **Geospatial Coordinates (4 cols)**:
   - `Restaurant_latitude`, `Restaurant_longitude`: GPS location of the restaurant.
   - `Delivery_location_latitude`, `Delivery_location_longitude`: GPS location of the customer.
4. **Timestamps (3 cols)**:
   - `Order_Date`: Date of order (`11-02-2022` to `06-04-2022`).
   - `Time_Orderd`: Time the customer placed the order.
   - `Time_Order_picked`: Time the rider picked up the food from the kitchen.
5. **Operational & Environmental Context + Target (9 cols)**:
   - `Weather_conditions`: `Sunny`, `Cloudy`, `Fog`, `Stormy`, `Sandstorms`, `Windy`.
   - `Road_traffic_density`: `Low`, `Medium`, `High`, `Jam`.
   - `Vehicle_condition`: `0` (Poor), `1` (Fair), `2` (Good).
   - `Type_of_order`: `Snack`, `Meal`, `Drinks`, `Buffet`.
   - `Type_of_vehicle`: `motorcycle`, `scooter`, `electric_scooter`, `bicycle`.
   - `multiple_deliveries`: Number of additional batched orders carried on the same trip (`0`, `1`, `2`, `3`).
   - `Festival`: Whether the order occurred on a festival day (`Yes` / `No`).
   - `City`: Urbanization tier (`Metropolitian`, `Urban`, `Semi-Urban`).
   - `Time_taken (min)`: **Target variable** — actual total minutes from order placement to doorstep delivery (`10` to `54` minutes).

---

### 3. What Was Wrong with the Raw Data? (Your Core EDA Story)
If you feed `zomato_delivery_dataset.csv` directly into Power BI without cleaning it in Python first, **five major bugs break your analysis**:

| # | Data Quality Issue | Exact Numbers Found | Why It Breaks Business Analytics | How We Fixed It in `Zomato_Delivery_EDA_and_Cleaning.ipynb` |
| :- | :--- | :--- | :--- | :--- |
| **1** | **String `'NaN'` Literals & Missing Values** | **9,131 missing cells** across 8 columns (`4,225` rows / `9.27%` of dataset): `Ratings` (1,908), `Age` (1,854), `Time_Orderd` (1,731), `City` (1,200), `multiple_deliveries` (993), `Weather` (616), `Traffic` (601), `Festival` (228). | Dropping rows with nulls would throw away **9.3% of Zomato's order volume** (`4,225` orders). Leaving `'NaN'` strings turns numeric columns into text in Power BI. | Converted `'NaN'` strings to true nulls, stripped whitespace, and used **context-aware imputation** (preserving 100% of `45,584` orders). |
| **2** | **Corrupted Logging Batch (`Vehicle_condition = 3`)** | **520 rows** had `Vehicle_condition = 3` (valid scale is `0, 1, 2`). All 520 rows were missing `Weather`, `Traffic`, and `Time_Orderd`. Inside this exact batch, **53 rows had impossible `Rating = 6.0` & `Age = 50`**, and **38 rows had underage `Age = 15` & `Rating = 1.0`**. | A `6.0` star rating on a 5-star scale skews average rider ratings; underage `15` is illegal for motor riders; code `3` creates a fake 4th bar on vehicle condition charts. | Replaced `Age in {15, 50}` with median (`30`), `Rating = 6.0` (and corrupted `1.0`) with median (`4.7`), and `Vehicle_condition = 3` with mode (`1`). |
| **3** | **Geospatial Corruption: "Null Island" `(0,0)` & Negative Sign Flips** | **3,640 rows** had restaurant GPS at `(0.0, 0.0)` (Null Island in the Atlantic Ocean!) while customer GPS had small numbers like `(0.08, 0.08)`. Another **431 rows** had negative latitudes/longitudes (e.g., `-27.16°` instead of `+27.16°` for Agra). | India lies entirely in the Northern & Eastern Hemispheres (`8°N–35°N`, `68°E–90°E`). Calculating distance between `-27.16°` and `+27.23°` produces **6,884 km** delivery distances! | Applied `abs()` to fix negative sign flips. For `(0,0)` rows, recognized `(0.08, 0.08)` as the **relative offset** $(\Delta\text{lat}, \Delta\text{lon})$, looked up the restaurant's true base coordinates via `Restaurant_ID` / `City_Code`, and added the offset. Distance now ranges cleanly from **1.47 km to 20.97 km** (mean **9.70 km**). |
| **4** | **Mixed Timestamp Formats & Midnight Rollover** | **4,068** `Time_Orderd` values and **4,127** `Time_Order_picked` values were exported from Excel as **decimal fractions of a day** (e.g., `0.458333333` instead of `11:00`, `0.75` instead of `18:00`). Also, orders at `23:50` picked up at `00:05` gave negative `-1,425 min` wait times. | Text/decimal mixing prevents calculating kitchen preparation delay (`Pickup_Delay`). | Converted all decimal fractions (`fraction × 1440` mins) into `HH:MM`, handled midnight rollover (`+1440` mins), and recovered clean **5, 10, and 15 minute** kitchen pickup delays (mean **9.99 min**). |
| **5** | **Missing Context in `Traffic`, `Weather`, and `City`** | `1,200` missing `City` values, `601` missing `Traffic`, `616` missing `Weather`. | Creates `"Blank"` categories in Power BI slicers and bar charts. | Imputed `City` using the exact `City_Code` prefix from `Delivery_person_ID`, `Traffic` using the order's time-of-day rush regime (`Jam` at 19:00–22:30 dinner peak, `High` at lunch, etc.), and `Weather` by `(City_Code, Order_Date)` mode. |

---

### 4. What Features Did We Engineer for the Dashboard? (12 New Columns -> 32 Total)
To turn raw logs into **Business Outcomes**, we engineered **12 new columns** in `zomato_delivery_cleaned.csv`:
1. **`Order_Day`, `Order_Month`, `Order_DayOfWeek`**: Calendar breakdowns from `Order_Date`.
2. **`Order_Time` & `Pickup_Time`**: Decimal hour of day (e.g., `21.92` = 9:55 PM) for rush-hour profiling.
3. **`Pickup_Delay` (`Pickup Delay(min)` in Power BI)**: Minutes spent waiting at the restaurant kitchen before the rider departs (`5`, `10`, or `15` mins; average **9.99 min**).
   - *Why it matters*: Allows us to split Total Delivery Time (**26.29 min**) into **Kitchen Prep Wait (9.99 min / 38%)** vs. **Road Transit Time (16.30 min / 62%)**!
4. **`Distance_km` (`Distance km` in Power BI)**: Exact Haversine great-circle distance in kilometers between restaurant and customer (**1.47 km to 20.97 km**, average **9.70 km**).
5. **`Actual_Time` (`Actual Delivery(min)`)**: Clean integer target (`Time_taken (min)`).
6. **`Predicted_Time` (`Predicted Delivery(min)`)**: Multi-factor predictive ETA benchmark (`R² = 0.8113`, `MAE = 3.18 min`, average **26.25 min**).
7. **`Error` & `Absolute_Error`**: `Actual_Time - Predicted_Time` and `|Error|` to measure ETA accuracy.
8. **`Delay_Risk`**: Operational SLA segmentation:
   - **`Low` (`< 22 min`)**: `15,749` orders (**34.55%**) — Fast deliveries.
   - **`Medium` (`22–30 min`)**: `16,233` orders (**35.61%**) — Within SLA but approaching the 30-min ceiling.
   - **`High` (`> 30 min` SLA Breach)**: `13,602` orders (**29.84%**) — Breached the 30-minute SLA promise.

---

### 5. Why Did We Redesign the Power BI Dashboard? (Old vs. New Comparison)
When explaining why you created a new dashboard while leaving the original `Delivery overview` page untouched, highlight these concrete improvements:
- **Broken Data Source Fixed**: The original semantic model pointed to a hardcoded path on another machine (`C:\Users\Ganesh\Downloads\APA project\Zomato_PowerBI_Predictions.csv`). We re-pointed it to our validated [`zomato_delivery_cleaned.csv`](file:///c:/Users/Rohith/Downloads/BI%20CaseStudy/zomato_delivery_cleaned.csv) in the workspace.
- **Visual & Labeling Errors Fixed**: In the original dashboard, the KPI card for `Avg Actual Delivery` was mislabeled *"Average Delivery Time by Weather"*, visuals had thick `9px` borders touching each other with zero margin, an unlinked *"Back"* button sat in the top-left corner, and slicers occupied a huge block at the bottom.
- **Upgraded from Descriptive Charts to a 2-Page Executive & Operational Architecture**:
  - **Page 1 (`Executive SLA & Business Outcomes`)**: Focuses on **external & strategic SLA performance** — tracking `Total Orders (45,584)`, `SLA Compliance % (70.16%)`, `High Risk / SLA Breach % (29.84%)`, `Avg Actual Delivery (26.29m)`, and `Avg Predicted ETA (26.25m)` across **Traffic Density**, **SLA Delay Risk Tiers**, **Weather Conditions**, and **City Tiers**.
  - **Page 2 (`Fleet & Operational Drivers`)**: Focuses on **controllable internal levers** that operations managers can act on today — `Active Riders (1,320)`, `Avg Distance (9.70 km)`, `Kitchen Pickup Delay (9.99m)` vs. `Road Transit Time (16.30m)`, `Avg Rider Rating (4.64 ★)`, **Vehicle Type Efficiency**, **Multi-Order Batching Penalty**, **Vehicle Maintenance Condition**, and **Order Category Breakdown**.

---

## PART 2: Step-by-Step Live Demonstration Pitch Script (5–7 Minutes)

Use this structured script during your presentation. It flows naturally from **Business Problem $\rightarrow$ Jupyter Notebook (EDA & Cleaning) $\rightarrow$ Power BI Dashboard $\rightarrow$ Business Recommendations**.

---

### Phase 1: The Hook & Business Objective (45 Seconds)
> *"Hello everyone. Today I am walking you through an end-to-end Business Intelligence case study on **Zomato's Last-Mile Delivery Operations**, covering **45,584 orders across 22 Indian cities**.*
>
> *Instead of jumping straight from raw CSV data into charts—which often leads to 'garbage in, garbage out' decisions—I structured this project around a real operational outcome: **Zomato's 30-Minute Delivery SLA (Service Level Agreement)**.*
>
> *Our goal is twofold:*
> *1. First, in Python (`Zomato_Delivery_EDA_and_Cleaning.ipynb`), audit and fix hidden data corruption so we don't lose 9.3% of our order volume or miscalculate delivery distances.*
> *2. Second, in Power BI, transform that cleaned dataset into a two-stage **Executive & Operational Dashboard** that separates external bottlenecks (like traffic jams and weather) from internal levers Zomato can directly control (like multi-order batching, kitchen prep wait times, and fleet maintenance)."*

---

### Phase 2: Walkthrough of the Jupyter Notebook (`Zomato_Delivery_EDA_and_Cleaning.ipynb`) (2.5 Minutes)
*(Open [`Zomato_Delivery_EDA_and_Cleaning.ipynb`](file:///c:/Users/Rohith/Downloads/BI%20CaseStudy/Zomato_Delivery_EDA_and_Cleaning.ipynb) on your screen and scroll through Stages 1 to 7 as you speak)*

1. **Stage 1 & 2 — Loading & Primary Key Integrity**:
   > *"In Stages 1 and 2, we load `zomato_delivery_dataset.csv` with 45,584 rows and 20 columns. First, I noticed missing values were stored as literal text strings `'NaN'` with extra spaces, so we standardized them and verified primary key integrity: **0 duplicate rows** and **45,584 unique Order IDs** across **1,320 delivery riders** and **440 partner restaurants**."*

2. **Stage 3 — The Missing Values Dilemma**:
   > *"Scrolling to Stage 3, our audit found **9,131 missing values across 8 columns**, affecting **4,225 rows—or 9.27% of all orders**. If we simply ran `df.dropna()`, we would wipe out nearly 10% of Zomato's business volume and underreport total orders in Power BI. Instead, I analyzed how missing values co-occurred."*

3. **Stage 4 — Uncovering the 4 Major Data Inconsistencies (The 'Aha!' Moment)**:
   > *"In Stage 4, deeper exploratory analysis revealed four critical inconsistencies that weren't just blank cells:*
   > - *First, **a corrupted logging batch of 520 rows** where `Vehicle_condition` was coded as `3` (on a `0, 1, 2` scale). Every single one of those 520 rows was missing weather, traffic, and order time—and inside that exact batch, **53 rows had an impossible 6.0-star rider rating with Age 50**, and **38 rows had an underage rider Age of 15**.*
   > - *Second, **Geospatial Coordinate Corruption**: **3,640 orders** had restaurant coordinates at `(0.0, 0.0)`—'Null Island' in the Atlantic Ocean—and **431 orders** had negative latitude or longitude sign-flips in the Southern Hemisphere. Without cleaning this, Haversine distance calculations produced **6,884 km** delivery trips!*
   > - *Third, **Mixed Time Formats**: Over **4,000 timestamps** in `Time_Orderd` and `Time_Order_picked` were exported as Excel decimal fractions of a day—like `0.458333` for `11:00 AM` or `0.75` for `6:00 PM`—along with midnight rollover where orders placed at `23:50` and picked up at `00:05` appeared to have negative wait times."*

4. **Stage 5 & 6 — Context-Aware Cleaning & Feature Engineering**:
   > *"In Stages 5 and 6, rather than guessing or dropping rows, we used domain logic encoded in the dataset itself:*
   > - *Notice that `Delivery_person_ID` starts with the city and restaurant code (like `RANCHIRES02DEL01`). And for the 3,640 `(0, 0)` rows, the customer latitude/longitude columns actually stored the exact coordinate offset $(\Delta\text{lat}, \Delta\text{lon})$. By looking up each restaurant's true base coordinates and adding that offset, we recovered 100% of the map coordinates and brought Haversine delivery distances into a realistic **1.47 km to 20.97 km** range (averaging **9.70 km**).*
   > - *Converting the decimal timestamps and handling midnight rollover revealed that kitchen pickup delays are cleanly distributed across **5, 10, and 15 minutes** (averaging **9.99 minutes**).*
   > - *Finally, in Stage 7, we engineered our SLA and predictive metrics (`Predicted_Time` with **R² = 0.8113** and **MAE = 3.18 min**, plus `Delay_Risk`) and exported the clean, 32-column dataset to `zomato_delivery_cleaned.csv` right in our project directory."*

---

### Phase 3: Walkthrough of the Power BI Dashboard (2.5 Minutes)
*(Switch to Power BI Desktop showing [`Zomato_Business_Dashboard.pbip`](file:///c:/Users/Rohith/Downloads/BI%20CaseStudy/Zomato_Business_Dashboard.pbip) or open [`Zomato_Executive_Dashboard.html`](file:///c:/Users/Rohith/Downloads/BI%20CaseStudy/Zomato_Executive_Dashboard.html))*

1. **Executive Context & Upgrade from Raw Legacy Layout**:
   > *"In Power BI, I upgraded the semantic model connection to point to our validated dataset `zomato_delivery_cleaned.csv`, eliminated the cluttered legacy layout, and built a structured two-page dashboard designed around how operations leaders actually make decisions."*

2. **Tab 1: `Executive SLA & Business Outcomes` (The 'What & Where')**:
   > *"On Page 1—**Executive SLA & Business Outcomes**—the top scorecard immediately answers how the business is performing against the 30-minute SLA across all **45,584 validated orders**:*
   > - *Our average delivery time is **26.29 minutes** (against a predicted ETA of **26.25 minutes**).*
   > - ***70.16%** of orders achieve **On-Time SLA Compliance ($\le 30\text{ min}$)**, while **29.84% (13,602 orders)** fall into **High Delay Risk ($> 30\text{ min}$)**.*
   > - *Looking at the four main visuals and using our left-hand slicer rail:*
   >   - ***Traffic Density (Top-Left)***: *In `Low` traffic, delivery takes just **21.3 minutes** (only 7.9% SLA breach). During evening `Jam` traffic—which accounts for **31.5% of our total order volume**—average delivery time jumps by 10 minutes to **31.2 minutes**, pushing the SLA breach rate over **50.6%**.*
   >   - ***Weather Impact (Bottom-Left)***: *`Fog` (**28.9 min**) and `Cloudy` (**28.9 min**) weather add **+7 minutes** per order compared to `Sunny` skies (**21.9 min**).*
   >   - ***Regional City Tiers (Bottom-Right)***: *`Urban` zones average **23.0 minutes**, `Metropolitian` zones average **27.1 minutes**, whereas `Semi-Urban` deliveries average **49.7 minutes** (a 100% SLA breach rate due to long road transit times)."*

3. **Tab 2: `Fleet & Operational Drivers` (The 'How We Fix It')**:
   > *"Traffic and weather are external factors Zomato cannot control. That's why I created Page 2—**Fleet, Batching & Operational Drivers**—which isolates the **internal levers** operations managers can control:*
   > - *First, look at the **KPI Row**: out of the **26.29-minute** total delivery time, **9.99 minutes (38%) is spent waiting at the restaurant kitchen (`Avg Pickup Delay`)**, and only **16.30 minutes (62%) is spent on the road (`Avg Transit Time`)** covering an average distance of **9.70 km**.*
   > - *Second, look at **Multi-Order Batching Penalty (Top-Right)**: When a rider carries a single order (`0` extra batched deliveries), delivery takes **22.9 minutes**. With `1` batched order, it stays within SLA at **26.7 minutes**. But when the dispatch algorithm stacks **2 or 3 batched orders** onto one rider, delivery time spikes to **40.5 minutes** and **47.8 minutes**—causing a **100% SLA breach rate**!*
   > - *Third, look at **Fleet Maintenance & Vehicle Type (Bottom-Left & Top-Left)**: Riders on `Poor Condition (0)` vehicles take **30.1 minutes** on average compared to **24.4 minutes** for `Fair (1)` or `Good (2)` vehicles—a **+5.6-minute delay caused purely by vehicle maintenance**. Furthermore, `Scooters` (**24.5 min**) and `Electric Scooters` (**24.5 min**) beat standard `Motorcycles` (**27.6 min**) by **3.1 minutes** in congested city corridors."*

---

### Phase 4: Closing with Actionable Business Recommendations (45 Seconds)
> *"To wrap up, this structured workflow from Python EDA to Power BI gives Zomato three immediate, data-backed operational actions to cut the 29.84% SLA breach rate:*
> 1. **Smart Dispatch Batching Cap**: *Cap multi-order batching at a maximum of **1 additional order** during `Jam` traffic, `Fog/Cloudy` weather, or `Festival` days—eliminating the 40–48 minute delivery spikes.*
> 2. **Fleet Maintenance & EV Shift**: *Require maintenance servicing for the 32.9% of rides currently operating on `Condition 0 (Poor)` vehicles (saving **5.6 minutes per order**), and prioritize `Scooters` and `Electric Scooters` for dense metro short-haul routes.*
> 3. **Kitchen Handoff Optimization**: *Since kitchen pickup delay accounts for **10 minutes (38% of total delivery time)** across all meal types, delaying rider dispatch notification by 3–4 minutes for slower kitchens will improve rider utilization and reduce doorstep wait times.*
>
> *Thank you! I'd be happy to take any questions or filter the live dashboard to explore any specific city tier or scenario."*

---

## PART 3: Quick Q&A Cheat Sheet (For Questions From Your Evaluators/Audience)

1. **Q: "Why didn't you just drop the rows with missing values in Python?"**
   - **Answer**: *"4,225 rows (9.27% of the dataset) had at least one missing field, but every single one of those rows had a valid Order ID, Pickup Time, and Actual Delivery Time (`Time_taken (min)`). Dropping them would underreport Total Orders by 4,225 in Power BI. Instead, we used domain-based imputation—such as extracting the city from `Delivery_person_ID`, imputing `Order_Time` from `Pickup_Time - 10 min`, and imputing traffic from the time-of-day rush hour."*

2. **Q: "How did you fix the `(0, 0)` GPS coordinates without making up random numbers?"**
   - **Answer**: *"When we inspected the 3,640 rows where `Restaurant_latitude` and `Restaurant_longitude` were `0, 0`, we noticed `Delivery_location_latitude` and `Delivery_location_longitude` had values like `0.06, 0.06`—which is the exact degree offset $(\Delta\text{lat}, \Delta\text{lon})$ from the restaurant to the customer! Since `Delivery_person_ID` encodes the exact restaurant (e.g., `RANCHIRES02`), we looked up that restaurant's median valid GPS coordinates from other orders and added the customer offset. That preserved both the exact delivery distance and the true city location."*

3. **Q: "Why did some timestamps look like `0.458333333` in the raw CSV?"**
   - **Answer**: *"In Excel, time is stored internally as a fraction of a 24-hour day (`11 hours / 24 hours = 0.458333333`, `18 hours / 24 hours = 0.75`). Multiply that decimal by `1,440` minutes in a day (`0.458333333 × 1440 = 660 minutes = 11:00`), and you recover the exact `HH:MM` timestamp."*

4. **Q: "What is the biggest controllable driver of SLA breaches?"**
   - **Answer**: *"Multi-order batching (`2` or `3` deliveries per trip, which causes 100% SLA breach at 40.5–47.8 mins) and `Poor (0)` vehicle condition (which adds +5.6 mins over `Fair/Good` vehicles, raising SLA breaches from 24.0% to 41.7%)."*
