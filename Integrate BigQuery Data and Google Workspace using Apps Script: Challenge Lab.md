## Task 1 - Apps Script: BigQuery -> Sheet
1. New tab: **script.google.com** > **New project**. Rename it (any name, e.g. `bq-sheets`).
2. Delete everything in the editor, paste the **Code.gs sample from the lab page**
   (it already contains your project ID in `var PROJECT_ID = "..."`; check it is not empty).
3. In the left file list: **Code.gs > ⋮ > Rename** to **`bq-sheets`** (it becomes bq-sheets.gs) > Enter.
4. Left side **Services (+)** > **BigQuery API** > **Add**.
5. **Save** (Ctrl+S). In the function dropdown pick **runQuery** > **Run**.
6. **Review permissions** > choose the student account > **Advanced** >
   **Go to ... (unsafe)** > **Allow**.
7. The log shows a spreadsheet URL: "Most common words in all of Shakespeare's works".
   -> **Check my progress** (Task 1).
   - Error "BigQuery API has not been enabled": remove the BigQuery service, add it again, Run.

## Task 2 - Connected Sheet formulas  (25 pts)
1. **sheets.google.com** > **Blank spreadsheet**.
2. **Data > Data connectors > Connect to BigQuery** > **Get connected** if asked.
3. Pick your **project** > **Public datasets** > **chicago_taxi_trips** > **taxi_trips** > **Connect**.
4. A tab **taxi_trips** opens. Click the **+** (bottom left) to add a normal sheet (**Sheet1**),
   then type these in Sheet1:
   - **A1:** `=COUNTUNIQUE(taxi_trips!company)`  (number of taxi companies)
   - **A2:** `=COUNTIF(taxi_trips!tips,">0")`  (rides with a tip)
   - **A3:** `=COUNTIF(taxi_trips!fare,">0")`  (trips with fare > 0)
   - **A4:** `=A2/A3`  then **Format > Number > Percent**  (percentage of rides with a tip)
5. For each formula cell click **Apply** (or **Refresh**) when Sheets asks, and wait until numbers appear.
   -> **Check my progress** (Task 2).

## Task 3 - Charts on the Connected Sheet  (25 pts)
Go back to the **taxi_trips** tab. For each chart: click **Chart** (top of the taxi_trips tab).

**Chart 1 - Pie: forms of payment**
- Chart type **Pie chart**
- **Label:** `payment_type`
- **Value:** `fare` > aggregation **COUNT**
- **Apply**.

**Chart 2 - Line: mobile payment revenue over time**
- **Chart** again > type **Line chart**
- **X-axis:** `trip_start_timestamp` > **Group by: Year-Month**
- **Series:** `fare` (SUM)
- **Filter:** `payment_type` > Filter by values > tick only **Mobile** > OK
- **Apply**.

**Chart 3 - Line: mobile payments since the 2015 peak**
- **Chart** again > **Line chart**
- **X-axis:** `trip_start_timestamp` > **Group by: Year**
- **Series:** `fare` (SUM)
- **Filter:** `payment_type` = **Mobile**, and add filter `trip_start_timestamp`
  > Filter by condition > **Date is after** > exact date **12/31/2014**
- **Apply**.

Wait until all three charts show data -> **Check my progress** (Task 3).

## Task 4 - New sheet with an address  (25 pts)
1. **sheets.google.com** > **Blank spreadsheet**.
2. Click **A1**, type `76 9th Ave, New York`, press **Enter**.
-> **Check my progress** (Task 4).

## If a check stays grey
- Wait 1-2 minutes (Drive changes reach the checker with a small delay) and click again.
- Make sure everything was done while signed in as the **student** account (top-right avatar).
- Connected Sheet results must be calculated: no "Apply" buttons left waiting on cells or charts.
