## 🛠 Tools & Technology

### 📊 Power BI
- Built 2-page interactive dashboard with slicers for Region, Department, and Date
- DAX measures for KPIs: Total Hires, Offer Accept Rate, Avg Time to Fill, 
  Hiring Risk Index, Aging Buckets
- Power Query used for data transformation and column profiling

---

### 🐍 Python (Data Preparation)
- Library: `pandas`, `google-generativeai`, `json`
- Loaded and cleaned raw hiring data from Excel
- Aggregated key metrics by department:
  - Open requisitions count
  - Aged 60+ and 90+ requisition counts
  - Average days to fill per department
- Exported cleaned summary as structured input for Gemini AI prompt

```python
import pandas as pd
import google.generativeai as genai

# Load hiring data
df = pd.read_excel("hiring_data.xlsx")

# Aggregate department risk metrics
dept_summary = df.groupby("Department").agg(
    open_reqs=("ReqID", "count"),
    aged_60=("DaysOpen", lambda x: (x >= 60).sum()),
    aged_90=("DaysOpen", lambda x: (x >= 90).sum()),
    avg_days_to_fill=("DaysToFill", "mean")
).reset_index()
```

---

### 🤖 Gemini AI Integration
- Used `google-generativeai` Python library to call Gemini API
- Passed aggregated hiring metrics as structured context in the prompt
- Prompted Gemini to return:
  - Executive Summary
  - Departments at Risk
  - Anomalies detected
  - Likely Business Causes
  - Recommended Management Actions

```python
genai.configure(api_key="YOUR_API_KEY")
model = genai.GenerativeModel("gemini-pro")

prompt = f"""
You are an HR analytics executive advisor.
Analyze the following hiring data and return:
1. Executive Summary
2. Departments at Risk
3. Anomalies
4. Likely Business Causes
5. Recommended Actions

Data:
{dept_summary.to_string()}

Respond in a structured, executive-ready format.
"""

response = model.generate_content(prompt)
print(response.text)
```

- AI output was copied into Power BI as a **static text visual** 
  inside the AI Insight page
- Risk index table was manually structured from AI output into 
  a reference table loaded via Power Query

---

### 🗄 SQL (Data Validation)
- Used **DB Browser for SQLite** to validate Python-cleaned data
- Cross-checked aggregated metrics against raw records

```sql
-- Total hires YTD
SELECT COUNT(*) AS total_hires
FROM hiring_data
WHERE hire_date BETWEEN '2024-01-01' AND '2024-12-31';

-- Open requisitions by department
SELECT department, COUNT(*) AS open_reqs
FROM hiring_data
WHERE status = 'Open'
GROUP BY department
ORDER BY open_reqs DESC;

-- Aging requisitions over 60 days
SELECT department,
       COUNT(*) AS aged_60_plus
FROM hiring_data
WHERE status = 'Open'
  AND days_open >= 60
GROUP BY department;

-- Average days to fill by department
SELECT department,
       ROUND(AVG(days_to_fill), 1) AS avg_days_to_fill
FROM hiring_data
WHERE status = 'Filled'
GROUP BY department
ORDER BY avg_days_to_fill DESC;

-- Hiring Risk Index (open reqs x2 + aged60 x3 + aged90 x5)
SELECT department,
       COUNT(*) * 2 + 
       SUM(CASE WHEN days_open >= 60 THEN 3 ELSE 0 END) +
       SUM(CASE WHEN days_open >= 90 THEN 5 ELSE 0 END) 
       AS hiring_risk_index
FROM hiring_data
WHERE status = 'Open'
GROUP BY department
ORDER BY hiring_risk_index DESC;
```
