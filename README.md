# Manufacturing SPC Alert System

> **Statistical Process Control (SPC) using SQL window functions to monitor manufacturing quality in real time.**

---

## Project Overview

This project analyzes historical manufacturing data to determine whether a production process is operating within acceptable control limits. Using **Statistical Process Control (SPC)** methodology, it flags any parts whose height measurements fall outside a dynamically computed acceptable range — enabling timely process adjustments and consistent product quality.

---

## The Problem

Manufacturing processes need constant monitoring. Rather than reacting to every small variation, SPC provides a data-driven framework:

- Only adjust the process when measurements fall **outside** statistically defined limits
- Use rolling historical data to define what "normal" looks like
- Flag anomalies automatically with a boolean `alert` column

---

## Control Limit Formulas

The acceptable range is defined by an **Upper Control Limit (UCL)** and a **Lower Control Limit (LCL)**:

$$UCL = \bar{x} + 3 \cdot \frac{\sigma}{\sqrt{5}}$$

$$LCL = \bar{x} - 3 \cdot \frac{\sigma}{\sqrt{5}}$$

Where:
- $\bar{x}$ = rolling average height (window of 5)
- $\sigma$ = rolling standard deviation of height (window of 5)
- $\sqrt{5}$ = square root of the window size

---

## Dataset

**Table:** `manufacturing_parts`

| Column | Description |
|--------|-------------|
| `item_no` | Unique item identifier |
| `length` | Length of the manufactured part |
| `width` | Width of the manufactured part |
| `height` | Height of the manufactured part *(key metric)* |
| `operator` | The operating machine that produced the part |

---

## SQL Solution

```sql
SELECT
    operator,
    row_number,
    height,
    avg_height,
    stddev_height,
    avg_height + 3 * stddev_height / SQRT(5) AS ucl,
    avg_height - 3 * stddev_height / SQRT(5) AS lcl,
    height NOT BETWEEN (avg_height - 3 * stddev_height / SQRT(5)) 
                   AND (avg_height + 3 * stddev_height / SQRT(5)) AS alert
FROM (
    SELECT
        operator,
        item_no,
        height,
        ROW_NUMBER() OVER (PARTITION BY operator ORDER BY item_no) AS row_number,
        AVG(height) OVER (
            PARTITION BY operator ORDER BY item_no 
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ) AS avg_height,
        STDDEV(height) OVER (
            PARTITION BY operator ORDER BY item_no 
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ) AS stddev_height,
        COUNT(*) OVER (
            PARTITION BY operator ORDER BY item_no 
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ) AS window_size
    FROM manufacturing_parts
) subquery
WHERE window_size = 5
ORDER BY item_no;
```

---

## Output

The final query (saved as `alerts` DataFrame) returns exactly these columns:

| Column | Type | Description |
|--------|------|-------------|
| `operator` | string | The machine/operator ID |
| `row_number` | integer | Sequential row within operator group |
| `height` | float | Measured height of the part |
| `avg_height` | float | Rolling 5-row average height |
| `stddev_height` | float | Rolling 5-row standard deviation |
| `ucl` | float | Upper Control Limit |
| `lcl` | float | Lower Control Limit |
| `alert` | boolean | `TRUE` if height is outside UCL/LCL |

---

## Key SQL Concepts Used

| Concept | Purpose |
|---------|---------|
| `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` | Sequential numbering per operator |
| `AVG() OVER (ROWS BETWEEN 4 PRECEDING AND CURRENT ROW)` | Rolling 5-row window average |
| `STDDEV() OVER (ROWS BETWEEN 4 PRECEDING AND CURRENT ROW)` | Rolling 5-row window std deviation |
| `COUNT() OVER (...)` | Detect incomplete windows |
| Subquery + `WHERE window_size = 5` | Filter out rows with < 5 observations |
| `NOT BETWEEN` | Boolean alert flag |

## Project Structure

```
├── README.md               # This file
├── notebook.ipynb          # DataCamp workspace notebook
├── parts.csv               # Raw manufacturing data
└── manufacturing.jpg       # Project cover image
```

---

## Tags

`SQL` `Window Functions` `Statistical Process Control` `SPC` `Manufacturing` `Data Engineering` `DataCamp` `Quality Control`
