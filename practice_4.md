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
```python
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

### Python
```python
# rank the first three videos for each user based on watched time.
videos_watched['watching_order'] = videos_watched.groupby('user_id')['watched_at'].rank()
videos_watched = videos_watched.sort_values(by=['user_id', 'watching_order'])
videos_watched = videos_watched[videos_watched['watching_order'] <=3]
temp_df = videos_watched.groupby('video_id')['user_id'].count().reset_index()

# Create another dataframe from all the videos that users have watched as their first 3 videos
temp_df.columns = ['video_id', 'count'] 

# Rank the videos based on dense rank
temp_df['rank'] = temp_df['count'].rank(method='dense', ascending=False)
temp_df = temp_df.rename(columns={'count': 'n_in_first_3'})
# filter videos that are ranked top 3 
temp_df.loc[temp_df['rank'] <= 3, ['video_id', 'n_in_first_3']]
```

## Q3a. Actual vs Predicted Arrival Time
Calculate the 90th percentile difference between Actual and Predicted arrival time in minutes for all completed trips within the first 14 days of 2022.

### SQL
```sql
SELECT
  diff AS ninetieth_percentile 
FROM
  (
    SELECT
      NTILE(9) OVER ( 
    ORDER BY
      diff) AS percentile1,
      diff 
    FROM
      (
        SELECT
          ABS(TIMESTAMPDIFF(MINUTE, predicted_eta, actual_time_of_arrival)) AS diff 
        FROM
          trip_details 
        WHERE
          status = 'completed' 
          AND actual_time_of_arrival BETWEEN '2022-01-01' AND '2022-01-14' 
      )
      a 
  )
  b 
WHERE
  percentile1 = 9 LIMIT 1
```
### Python
```python

# filter out by date and status
trip_details = trip_details[(trip_details['status'] == 'completed') & (trip_details['actual_time_of_arrival'] <= '14-01-2022') & 
(trip_details['actual_time_of_arrival'] >= '01-01-2022')]

# time diff in absolute figure and get difference in minutes
trip_details['difference_pred'] = abs(trip_details['actual_time_of_arrival'] - trip_details['predicted_eta'])/60

# find 90th percentile
trip_details['difference_pred'].quantile(0.9)
```
