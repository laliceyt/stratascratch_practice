## Q1a. Most Recent Employee Login Details

Amazon's information technology department is looking for information on employees' most recent logins.
The output should include all information related to each employee's most recent login.

### SQL
```sql 
SELECT
  original_t.id,
  original_t.worker_id,
  DATE_FORMAT(original_t.login_timestamp, '%Y-%m-%d %H:%i:%s') AS login_timestamp,
  original_t.ip_address,
  original_t.country,
  original_t.region,
  original_t.city,
  original_t.device_type
FROM worker_logins original_t
JOIN (SELECT
  worker_id,
  MAX(login_timestamp) AS max_time
FROM worker_logins
GROUP BY worker_id) agg_t
  ON original_t.worker_id = agg_t.worker_id
  AND original_t.login_timestamp = agg_t.max_time
```

### Python
```python
# Import your libraries
import pandas as pd

# Start writing code
worker_logins.head()

# Find the last login of each worker (aka max time)
results_df = worker_logins.groupby('worker_id', as_index=False)['login_timestamp'].max()

results_df.columns = ['worker_id', 'last_login']

# merge the df with max time for each worker and original df
worker_logins = worker_logins.merge(results_df, left_on=['worker_id', 'login_timestamp'], right_on=['worker_id', 'last_login'])

# Reorder the columns according to expected output
cols = [1,-1,0] + list(range(2,worker_logins.shape[1]-1))
worker_logins.iloc[:,cols].sort_values(by="worker_id")
```
