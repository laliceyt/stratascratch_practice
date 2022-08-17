
## Q1a: Most Profitable Companies
Find the 3 most profitable companies in the entire world.
Output the result along with the corresponding company name.
Sort the result based on profits in descending order.

### SQL 

```sql
select
  company,
  profits
from forbes_global_2010_2014
ORDER BY profits DESC
LIMIT 3;
```

### Python

``` python
import pandas as pd

#sort order
df = forbes_global_2010_2014.sort_values(by='profits', ascending=False)

#get top 3
df.head(3)[['company', 'profits']]
```

## Q1b: Finding User Purchases
Write a query that'll identify returning active users. 
A returning active user is a user that has made a second purchase within 7 days of any other of their purchases. 
Output a list of user_ids of these returning active users.

### SQL 

```sql
select 
  DISTINCT a.user_id
from amazon_transactions a
JOIN
amazon_transactions b
ON a.user_id = b.user_id 
WHERE a.created_at - b.created_at BETWEEN 0 and 7 AND a.id != b.id
ORDER BY a.user_id;
```

### Python

``` python

import pandas as pd
import datetime

# sort values by user and date
df = amazon_transactions.sort_values(by=['user_id', 'created_at'])

# group by user and get diff btw 2 rows
df['difference'] = df.groupby('user_id')['created_at'].diff()

# remove duplicates
df[df['difference'] <= pd.Timedelta(days=7)]['user_id'].drop_duplicates()
```

## Q1c. Highest Cost Orders
Find the customer with the highest daily total order cost between 2019-02-01 to 2019-05-01. 
If customer had more than one order on a certain day, sum the order costs on daily basis. 
Output customer's first name, total cost of their items, and the date.

### SQL
```sql
WITH orders AS (select 
  c.first_name, 
  SUM(o.total_order_cost) as sum_order, 
  o.order_date
from customers c
JOIN orders o
ON c.id = o.cust_id
WHERE o.order_date BETWEEN '2019-02-01' AND '2019-05-01' 
GROUP BY 1,3)
SELECT * FROM orders
WHERE sum_order IN (SELECT max(sum_order) FROM orders);
```

### Python

``` python
import pandas as pd

order_df = orders[orders['order_date'].between('2019-02-01', '2019-05-01')]
order_df = order_df[['cust_id', 'order_date', 'total_order_cost']].groupby(['cust_id', 'order_date']).sum().reset_index()

df = customers.merge(order_df, left_on='id', right_on='cust_id')
df = df[['first_name', 'order_date', 'total_order_cost']]

#convert date with time to date
df['order_date'] = pd.to_datetime(df['order_date']).dt.date

# get index of max value
max_idx = df['total_order_cost'].idxmax()

# convert the row to column 
pd.DataFrame(df.iloc[max_idx]).T
```

## Q2a. Highest Target Under Manager
Find the highest target achieved by the employee or employees who works under the manager id 13. 
Output the first name of the employee and target achieved. 
The solution should show the highest target achieved under manager_id=13 and which employee(s) achieved it.

### SQL
``` sql
WITH mgr AS
(SELECT
  *
from salesforce_employees
WHERE manager_id = 13)

SELECT 
  first_name,
  target
FROM mgr
WHERE target = (SELECT MAX(target) FROM mgr);
```

### Python

``` python
import pandas as pd

# check if anyone is under more than 1 manager
result = salesforce_employees[['id', 'manager_id']].groupby(by='id').count() > 1
result.sum()

# filter those under manager id 13 and get names of the highest target (can be more than 1)
salesforce_employees[salesforce_employees['manager_id'] == 13].nlargest(1, 'target', keep='all').loc[:, ['first_name', 'target']]
```

## Q2b. Classify Business Type
Classify each business as either a restaurant, cafe, school, or other. 
A restaurant should have the word 'restaurant' in the business name. 
For cafes, either 'cafe', 'café', or 'coffee' can be in the business name. 
'School' should be in the business name for schools. All other businesses should be classified as 'other'. 
Output the business name and the calculated classification.


### SQL
``` sql
SELECT 
  DISTINCT business_name,
  CASE 
    WHEN LOWER(business_name) LIKE '%restaurant%' THEN 'restaurant'
    WHEN LOWER(business_name) ~ 'caf(e|é)|coffee' THEN 'cafe'
    WHEN LOWER(business_name) LIKE '%school%' THEN 'school'
    ELSE 'other' END AS business_type
FROM sf_restaurant_health_violations
ORDER BY business_type, business_name
```


### Python

``` python
import pandas as pd
import re

def category(name):
    l_name = name.lower()
    if 'school' in l_name:
        return 'school'
    elif 'restaurant' in l_name:
        return 'restaurant'
    elif re.search('(caf(e|é)|coffee)', l_name):
        return 'cafe'
    else:
        return 'other'
        
sf_restaurant_health_violations['business_type'] = sf_restaurant_health_violations['business_name'].apply(category)
sf_restaurant_health_violations[['business_name', 'business_type']].sort_values(by=['business_type', 'business_name']).drop_duplicates()
```

## Q2c. Acceptance Rate By Date
What is the overall friend acceptance rate by date?
Your output should have the rate of acceptances by the date the request was sent. 
Order by the earliest date to latest.

### SQL
```sql
SELECT
  a.date,
  SUM(CASE WHEN b.action = 'accepted' THEN 1 ELSE 0 END)
  /SUM(CASE WHEN a.action = 'sent' THEN 1 ELSE 0 END)::decimal as percentage_acceptance
FROM fb_friend_requests a
LEFT JOIN fb_friend_requests b
ON a.user_id_sender = b.user_id_sender AND a.user_id_receiver = b.user_id_receiver AND a.date <> b.date
WHERE a.action = 'sent'
GROUP BY a.date;
```

### Python
```python
import pandas as pd

# filter by actions
send_df = fb_friend_requests[fb_friend_requests['action'] == 'sent']
accept_df = fb_friend_requests[fb_friend_requests['action'] == 'accepted']

# left join
df = send_df.merge(
    accept_df, on=['user_id_sender', 'user_id_receiver'], how='left')

# group by date
df = df[['date_x', 'action_x', 'action_y']].groupby('date_x').count().reset_index()

# get rate
df['acceptance_rate'] = df['action_y'] / df['action_x']

# format date
df['date'] = pd.to_datetime(df['date_x']).dt.date

df[['date', 'acceptance_rate']]
```
## Q3a. Highest Energy Consumption
Find the date with the highest total energy consumption from the Meta/Facebook data centers. 
Output the date along with the total energy consumption across all data centers.

### SQL
```sql
WITH total AS
(
  SELECT 
    *
  FROM fb_eu_energy eu
UNION ALL
  SELECT 
    *
  FROM fb_asia_energy asia
UNION ALL
  SELECT 
    *
  FROM fb_na_energy na),
  sum_table AS (
SELECT 
  date,
  SUM(consumption) AS total_energy
FROM total
GROUP BY date)

SELECT * FROM sum_table
WHERE total_energy = (SELECT MAX(total_energy) FROM sum_table);
```

### Python

``` python

import pandas as pd

# joining all three tables and sum them by date.
df = pd.concat([fb_eu_energy,fb_asia_energy,fb_na_energy]).groupby('date').sum().reset_index()

# convert datetime to date
df['date'] = pd.to_datetime(df['date'])
df['date'] = df['date'].dt.date
df

# find max
df.nlargest(1, 'consumption')[['date', 'consumption']]
```

## Q3b. Number of violations
You're given a dataset of health inspections. 
Count the number of violation in an inspection in 'Roxanne Cafe' for each year. 
If an inspection resulted in a violation, there will be a value in the 'violation_id' column. Output the number of violations by year in ascending order.

### SQL
```sql
SELECT
  EXTRACT(YEAR FROM inspection_date) as year,
  SUM(CASE WHEN violation_id is null THEN 0
  ELSE 1 END) AS n_inspections
FROM sf_restaurant_health_violations
WHERE business_name = 'Roxanne Cafe'
GROUP BY business_name, year
ORDER BY year;
```

### Python

``` python
import pandas as pd

# filter for roxanne cafe and violation_id that are not null
df = sf_restaurant_health_violations.loc[
    (sf_restaurant_health_violations['business_name'] == 'Roxanne Cafe') & (sf_restaurant_health_violations['violation_id'].notnull())]

# convert datetime to year
df['inspection_date'] = pd.to_datetime(df['inspection_date'])
df['inspection_date'] = df['inspection_date'].dt.year

# group by year and count no. of violations. Reset index
df.groupby(by=['inspection_date'])['violation_id'].count().reset_index()
```

## Q3c. Customer Revenue In March
Calculate the total revenue from each customer in March 2019. 
Include only customers who were active in March 2019.

### SQL
```sql
SELECT
  cust_id,
  SUM(total_order_cost) as revenue
from orders 
WHERE order_date >= '2019-03-01' and order_date < '2019-04-01'
GROUP BY cust_id
ORDER BY revenue DESC;
```

### Python

``` python
import pandas as pd

# filter for active customers in Mar 2019
df = orders.loc[(orders['order_date'] >= '2019-03-01') & (orders['order_date'] < '2019-04-01')]

# group id and sum up order cost
df = df.groupby(by='cust_id')['total_order_cost'].sum().reset_index()

#rename total_order_cost column to revenue
df = df.rename(columns={'total_order_cost': 'revenue'})

# sort results by total_order_cost in desc
df.sort_values(by='revenue', ascending=False)
