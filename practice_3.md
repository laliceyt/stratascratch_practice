## Q1a: Monthly Percentage Difference
Given a table of purchases by date, calculate the month-over-month percentage change in revenue. The output should include the year-month date (YYYY-MM) and percentage change, rounded to the 2nd decimal point, and sorted from the beginning of the year to the end of the year.
The percentage change column will be populated from the 2nd month forward and can be calculated as ((this month's revenue - last month's revenue) / last month's revenue)*100.

### SQL 

```sql
SELECT
  year_month,
  ROUND((revenue - LAG(revenue, 1) OVER (ORDER BY year_month))/LAG(revenue, 1) OVER (ORDER BY year_month)*100,2) as revenue_diff_pct
FROM
(select 
 to_char(created_at,'YYYY-MM') as year_month,
 SUM(value) as revenue
from sf_transactions
GROUP BY 1)a;;
```

> A better alternative (from the stratascratch) is to use join as my solution won't work if we have a gap in month.

```sql
with monthly_revenue as 
(select 
  to_char(created_at, 'yyyy-MM') yrmo, 
  to_date(to_char(created_at, 'yyyy-MM')||'-01', 'yyyy-MM-dd') month_beginning,
  sum(value) revenue 
  from sf_transactions 
  group by to_char(created_at, 'yyyy-MM')
 ) 
 select 
  t1.yrmo, 
  round((t1.revenue/t2.revenue-1)*100,2) 
 from monthly_revenue t1 
 left join monthly_revenue t2 
  on t1.month_beginning= (t2.month_beginning + interval '1 month') 
 order by t1.month_beginning;
```
