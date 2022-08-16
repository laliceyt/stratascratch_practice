
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


