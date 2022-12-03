## Q1a. Customer Tracking
Given the users' sessions logs on a particular day, calculate how many hours each user was active that day.
Note: The session starts when state=1 and ends when state=0.

### SQL
```sql 
SELECT
  cust_id,
  SUM(active_hours) FROM
(select
  *,
  TIME_TO_SEC(TIMEDIFF(
   timestamp,
    LAG(timestamp, 1) OVER (PARTITION BY cust_id ORDER BY cust_id, timestamp)
    ))/3600 AS active_hours
from cust_tracking
ORDER BY cust_id, timestamp) temp
WHERE state=0
GROUP BY cust_id;
```

### Python
``` python
# create a lag column that will be the start time
cust_tracking['start']= cust_tracking.groupby(['cust_id'])['timestamp'].shift(1)
cust_tracking = cust_tracking.rename(columns={'timestamp': 'end'})

# Filter out the rows with state = 0 as these are rows where status is off at end time.
filtered_df = cust_tracking[cust_tracking['state'] == 0]

# Get time difference between start and end time in hours
filtered_df['total_hours'] = (filtered_df['end'] - filtered_df['start']).dt.seconds /(60*60)

# Aggregate the hours by customer
filtered_df = filtered_df.groupby(['cust_id'], as_index=False)['total_hours'].sum()
```


## Q1b. Inactive Free Users
Return a list of users with status free who didn’t make any calls in Apr 2020.

### SQL
```sql 
SELECT user_id
FROM rc_users
WHERE user_id NOT IN
    (SELECT rc_calls.user_id
     FROM rc_calls
     JOIN rc_users ON rc_calls.user_id = rc_users.user_id
     WHERE DATE_FORMAT(date, '%m-%Y') = '04-2020')
  AND status = 'free'
ORDER BY user_id;
```

### Python
```
rc_calls.date= pd.to_datetime(rc_calls.date)

# get list of users who made calls in 04-2020
excluded_users = list(rc_calls[(rc_calls.date.dt.month == 4) & (rc_calls.date.dt.year == 2020)].user_id)

# exclude the list of users above. Get users who didn't make calls during the period and filter for those with free status
rc_users[~rc_users.user_id.isin(excluded_users) & (rc_users.status == 'free')].user_id
```
## Q2a. First Three Most Watched Videos
After a new user creates an account and starts watching videos, the user ID, video ID, and date watched are captured in the database. Find the top 3 videos most users have watched as their first 3 videos. Output the video ID and the number of times it has been watched as the users' first 3 videos.


In the event of a tie, output all the videos in the top 3 that users watched as their first 3 videos.

### SQL
```sql
WITH agg_func
AS (SELECT
  video_id,
  COUNT(user_id) AS counter
FROM (SELECT
  *,
  ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY watched_at) AS ranking
FROM videos_watched) temp
WHERE ranking <= 3
GROUP BY video_id
ORDER BY counter DESC),
func_2
AS (SELECT
  video_id,
  DENSE_RANK() OVER (ORDER BY counter DESC) AS ranking,
  counter
FROM agg_func)
SELECT
  video_id,
  counter AS n_in_first_3
FROM func_2
WHERE ranking <= 3;
```
