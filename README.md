# Nigerian Energy Access & Distribution Intelligence Dashboard

A 3-page Power BI dashboard analysing Nigeria’s electricity access trends, state-level disparities, and DISCO distribution performance using World Bank, NBS, and NERC-referenced data.


## Project Overview

Nigeria’s electricity crisis is one of the most documented yet underanalysed problems in Africa. With over 90 million people still without power, and distribution losses averaging 29% across all DISCOs, the data tells a story that goes beyond infrastructure — it reveals deep regional inequality and systemic inefficiency.

This project was built to demonstrate end-to-end Power BI skills including data sourcing, Power Query transformation, data modelling, DAX measure writing, and professional dashboard design.


## Tools Used

- Microsoft Power BI Desktop
- Microsoft Excel
- Power Query (M Language)
- DAX (Data Analysis Expressions)



## Data Sources

NGA_Electricity_Access == World Bank Open Data   ==   % of Nigerian population with electricity access (1990–2023)                               
NGA_Power_Consumption ==  World Bank Open Data   ==  Electric power consumption in kWh per capita (1990–2023)                                   
State_Access  == Structured manually from NBS/NERC published figures|State-level electricity access % including urban and rural breakdown for all 36 states + FCT
DISCO_Performance  == Structured manually from NERC Annual Report figures|DISCO-level allocation, metering, billing, collection, and losses data                      |

### Note on Data Collection

The World Bank datasets were downloaded as CSV files and contained global data for all countries Nigeria’s data was extracted during the Power Query transformation stage. The State_Access and DISCO_Performance tables were manually structured from published NBS and NERC figures, a deliberate choice that allowed full control over data quality and structure.



## Data Cleaning & Transformation (Power Query)

### NGA_Electricity_Access & NGA_Power_Consumption

Both World Bank files arrived in a wide format with years as column headers and junk rows at the top. The following steps were applied to both:

1. **Removed top 3 rows** — the raw file had metadata rows above the actual headers
1. **Used first row as headers** — promoted the correct row to column headers
1. **Filtered for Nigeria only** — both files contained all world countries; filtered Country Code = NGA
1. **Unpivoted year columns** — transformed the wide format (60+ year columns) into a long format with Year and Value columns. This is essential for Power BI time-based visuals
1. **Renamed columns** — Attribute → Year, Value → Access_Pct / Consumption_KWh
1. **Removed unnecessary columns** — dropped Indicator Name and Indicator Code
1. **Set correct data types** — Year as Whole Number, value columns as Decimal Number

### State_Access & DISCO_Performance

These were manually built in Notepad as CSV files and were already in a clean, structured format. Steps applied:

1. Verified data types for all columns
1. Confirmed text columns (State, Zone, DISCO) were set to Text
1. Confirmed numeric columns were set to correct types

### Key Learning

The unpivot step was the most important transformation in this project. World Bank data always comes in wide format — knowing how to unpivot is a non-negotiable Power Query skill for any analyst working with open data.


## Data Modelling

### Relationship Challenge

Initially attempted to connect NGA_Electricity_Access and NGA_Power_Consumption through their Year columns directly. This resulted in a Many-to-Many relationship warning because both tables had one row per year with no clear “one side.”

### Solution  Year Table

Created a central Year_Table using DAX:

```
Year_Table = GENERATESERIES(1990, 2023, 1)
```

This table contains one row per year and acts as the single “one side” connecting to all fact tables. All time-based tables (NGA_Electricity_Access, NGA_Power_Consumption, State_Access) connect to Year_Table through a One-to-Many relationship with single cross-filter direction.

DISCO_Performance has no year column and stands as a standalone table — appropriate since it represents a single period snapshot.

### Model Type

This model uses a shared dimension approach rather than a classic star schema, as there is no single central fact table. The Year_Table serves as the shared time dimension across multiple fact tables.


## DAX Measures

### NGA_Electricity_Access Measures

```dax
Avg Access Rate = ROUND(AVERAGE(NGA_Electricity_Access[Access_Pct]) / 100, 3)

Latest Access Rate = CALCULATE([Avg Access Rate], Year_Table[Year] = MAX(Year_Table[Year]))

YoY Access Change = 
VAR CurrentYear = MAX(Year_Table[Year])
VAR PreviousValue = CALCULATE([Avg Access Rate], Year_Table[Year] = CurrentYear - 1)
VAR CurrentValue = CALCULATE([Avg Access Rate], Year_Table[Year] = CurrentYear)
RETURN CurrentValue - PreviousValue
```

### NGA_Power_Consumption Measures

```dax
Avg Consumption = AVERAGE(NGA_Power_Consumption[Consumption_KWh])

Latest Consumption = CALCULATE([Avg Consumption], Year_Table[Year] = MAX(Year_Table[Year]))

YoY Consumption Change = 
VAR CurrentYear = MAX(Year_Table[Year])
VAR PreviousValue = CALCULATE([Avg Consumption], Year_Table[Year] = CurrentYear - 1)
VAR CurrentValue = CALCULATE([Avg Consumption], Year_Table[Year] = CurrentYear)
RETURN ROUND((CurrentValue - PreviousValue) / PreviousValue, 2)
```

### State_Access Measures

```dax
Top Access State = CALCULATE(SELECTEDVALUE(State_Access[State]), FILTER(State_Access, State_Access[Access_Pct] = MAXX(State_Access, State_Access[Access_Pct])))

Lowest Access State = CALCULATE(SELECTEDVALUE(State_Access[State]), FILTER(State_Access, State_Access[Access_Pct] = MINX(State_Access, State_Access[Access_Pct])))

Top Access Pct = ROUND(MAXX(State_Access, State_Access[Access_Pct]), 1) / 100

Lowest Access Pct = ROUND(MINX(State_Access, State_Access[Access_Pct]), 1) / 100

National Avg Access = ROUND(AVERAGE(State_Access[Access_Pct]), 1)

States Below Avg = COUNTROWS(FILTER(State_Access, State_Access[Access_Pct] < AVERAGE(State_Access[Access_Pct])))

North South Gap = 
VAR SouthAvg = CALCULATE(AVERAGE(State_Access[Access_Pct]), State_Access[Zone] IN {"South West", "South South", "South East"})
VAR NorthAvg = CALCULATE(AVERAGE(State_Access[Access_Pct]), State_Access[Zone] IN {"North West", "North East", "North Central"})
RETURN ROUND((SouthAvg - NorthAvg) / 100, 3)
```

### DISCO_Performance Measures

```dax
Avg Losses Pct = AVERAGE(DISCO_Performance[Losses_Pct])

Total Allocated MWh = SUM(DISCO_Performance[Allocated_MWh])

Collection Efficiency = AVERAGE(DISCO_Performance[Collection_Efficiency_Pct])

Worst DISCO = CALCULATE(SELECTEDVALUE(DISCO_Performance[DISCO]), FILTER(DISCO_Performance, DISCO_Performance[Losses_Pct] = MAXX(DISCO_Performance, DISCO_Performance[Losses_Pct])))

Worst DISCO Losses Pct = ROUND(MAXX(DISCO_Performance, DISCO_Performance[Losses_Pct]) / 100, 3)

Best DISCO = CALCULATE(SELECTEDVALUE(DISCO_Performance[DISCO]), FILTER(DISCO_Performance, DISCO_Performance[Losses_Pct] = MINX(DISCO_Performance, DISCO_Performance[Losses_Pct])))

Total Collected MWh = SUM(DISCO_Performance[Collected_MWh])
```

## Dashboard Pages

### Page 1 — National Energy Overview

**Visuals:** 4 KPI cards, Access Rate trend line chart, Power Consumption trend line chart, Electricity Access by Zone bar chart, Year slicer, Insight text panel

**Key Insight:** Nigeria’s electricity access grew from 27% in 1990 to 61% in 2023 — a 34% improvement over 30 years. However, over 90 million Nigerians remain without power, with the North East recording the lowest regional access at just 26%.

### Page 2 — State Analysis

**Visuals:** 4 KPI cards, Top 10 States bar chart, State breakdown matrix with conditional formatting (red to green gradient), Zone slicer,

**Key Insight:** Lagos leads with 87% access while Yobe sits at just 22%. The South West average (63%) is more than double the North East average (26%), revealing a stark regional electricity inequality.

### Page 3 — DISCO Performance

**Visuals:** 4 KPI cards, Distribution Losses by DISCO bar chart, Allocated vs Collected clustered bar chart, Full DISCO performance table with conditional formatting, Zone slicer

**Key Insight:** Yola DISCO records the highest distribution losses at 37.9% while Eko DISCO leads in collection efficiency at 94.2%. Across all 11 DISCOs, an average of 29% of allocated electricity is lost before it reaches paying customers — representing billions in annual revenue loss for the sector.


## Mistakes Made & How They Were Fixed

Used column instead of measure on visuals — showing SUM instead of average   == I Replaced column with the correct DAX measure          Always use measures on visuals, not raw columns                                                            
Many-to-Many relationship error when connecting Year columns directly I Created a central Year_Table using GENERATESERIES                 Every Power BI model handling time needs a dedicated date/year dimension                              
YoY measures returned blank — used SELECTEDVALUE which needs a filter context, I Replaced SELECTEDVALUE with MAX(Year_Table[Year])        SELECTEDVALUE only works when a slicer or filter is active                                                 
Access percentage measures showing 4806% after applying percentage format I Divided all percentage measures by 100 before formatting      Power BI percentage format multiplies by 100 — values stored as whole numbers must be divided first        
Map visual completely blank for Nigerian states, I Changed State column Data Category to State or Province, then added “, Nigeria” suffix to state names Power BI maps need geographic context Nigerian states are not natively recognised without country context
I Tried to connect two fact tables directly through Year, I Used Year_Table as the central dimension instead                          Many-to-Many relationships should always be resolved through a bridge/dimension table                      



## Key Insights

- Nigeria’s electricity access grew 34 percentage points over 30 years but growth has been slow and inconsistent
- Power consumption peaked at 219 kWh per capita in 2016 and has not recovered — reflecting ongoing grid instability
- The North-South electricity access gap stands at approximately 27 percentage points
- Yola DISCO loses 37.9% of all electricity allocated to it — the worst in the country
- Average distribution losses of 29% across all DISCOs represent a massive systemic revenue and infrastructure problem

-----

## Connect

Built by Ifechukwu Okoli Chinwe
[LinkedIn] = https://www.linkedin.com/in/ife-okoli
[Medium] = https://medium.com/@ifechukwuokoli97
