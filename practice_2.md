## Q1a: Host Popularity Rental Prices
You’re given a table of rental property searches by users. 
The table consists of search results and outputs host information for searchers. 
Find the minimum, average, maximum rental prices for each host’s popularity rating. The host’s popularity rating is defined as below:
0 reviews: New
1 to 5 reviews: Rising
6 to 15 reviews: Trending Up
16 to 40 reviews: Popular
more than 40 reviews: Hot

Tip: The id column in the table refers to the search ID. You'll need to create your own host_id by concating price, room_type, host_since, zipcode, and number_of_reviews.

### SQL
``` sql
SELECT
  host_pop_rating,
  MIN(price) AS min_price,
  AVG(price) AS avg_price,
  MAX(price) AS max_price
FROM
(select 
  DISTINCT CONCAT(price, room_type, host_since, zipcode, number_of_reviews) as host_id,
  price,
  number_of_reviews,
  CASE 
    WHEN number_of_reviews = 0 THEN 'New'
    WHEN number_of_reviews BETWEEN 1 AND 5 THEN 'Rising'
    WHEN number_of_reviews BETWEEN 6 AND 15 THEN 'Trending Up'
    WHEN number_of_reviews BETWEEN 16 AND 40 THEN 'Popular'
  ELSE 'Hot' END as host_pop_rating
from airbnb_host_searches
ORDER BY number_of_reviews) a
GROUP BY host_pop_rating
```

### Python
``` python
import pandas as pd

# create unique host id and remove duplicates
airbnb_host_searches['host_id'] = airbnb_host_searches['price'].astype(str) + airbnb_host_searches['zipcode'].astype(str) + airbnb_host_searches['room_type'] + airbnb_host_searches['host_since'].astype(str) + airbnb_host_searches['zipcode'].astype(str) + airbnb_host_searches['number_of_reviews'].astype(str)

df = airbnb_host_searches.drop_duplicates(['host_id'])

# categorise host based on no. of reviews
def host_rating(cnt):
    if cnt == 0:
        return 'New'
    elif cnt <=5:
        return 'Rising'
    elif cnt <= 15:
        return 'Trending Up'
    elif cnt <= 40:
        return 'Popular'
    else:
        return 'Hot'
        
df['host_popularity'] = df['number_of_reviews'].apply(host_rating)

# use aggregate function to get min, mean, max values of each host pop category
df.groupby('host_popularity', as_index=False)['price'].aggregate(['min', 'mean', 'max']).reset_index()
```
## Q1b: Top 5 States With 5 Star Businesses
Find the top 5 states with the most 5 star businesses. 
Output the state name along with the number of 5-star businesses and order records by the number of 5-star businesses in descending order. 
In case there are ties in the number of businesses, return all the unique states. If two states have the same result, sort them in alphabetical order.

### SQL
``` sql 
SELECT
  state,
  n_businesses FROM
(SELECT
  state,
  COUNT(business_id) as n_businesses,
  RANK() OVER (ORDER BY COUNT(business_id) DESC) as rank_cnt
FROM yelp_business
WHERE stars=5 
GROUP BY state) a
WHERE rank_cnt <= 5;
```

### Python
``` python
import pandas as pd

# filter out businesses with 5 stars
df = yelp_business.loc[yelp_business.stars == 5]

# get no. of businesses in each state with 5 stars
df = df.groupby(['state'], as_index=False)['business_id'].count()
df = df.rename(columns={'business_id': 'n_businesses'})
df
# Get the top 5 without dropping duplicates
df.nlargest(5, 'n_businesses', keep='all')
```

## Q1c: Top Cool Votes
Find the review_text that received the highest number of  'cool' votes.
Output the business name along with the review text with the highest numbef of 'cool' votes.

### SQL
``` sql
SELECT
  business_name,
  review_text
FROM yelp_reviews
WHERE cool = 
(select
  MAX(cool)
from yelp_reviews)
```

### Python
``` python
import pandas as pd

# filter out the reviews with max. no. of cool votes and show business name and review text.
yelp_reviews.loc[yelp_reviews.cool == yelp_reviews.cool.max(), ['business_name', 'review_text']]
```

## Q2a: Premium vs Freemium
Find the total number of downloads for paying and non-paying users by date. Include only records where non-paying customers have more downloads than paying customers. The output should be sorted by earliest date first and contain 3 columns date, non-paying downloads, paying downloads.

### SQL
``` sql
SELECT * 
FROM (select 
  date,
  SUM(CASE WHEN paying_customer = 'no' THEN downloads ELSE 0 END) AS non_paying,
  SUM(CASE WHEN paying_customer = 'yes' THEN downloads ELSE 0 END) AS paying
FROM ms_user_dimension u
JOIN ms_acc_dimension a
ON u.acc_id = a.acc_id
JOIN ms_download_facts dl
ON dl.user_id = u.user_id
GROUP BY date) t
WHERE non_paying > paying
ORDER BY date;
```

### Python
``` python
import pandas as pd

# merge all three tables
df = ms_user_dimension.merge(ms_acc_dimension).merge(ms_download_facts)

# reshape the data based on column values from paying customer.
df = df.groupby(['date', 'paying_customer'], as_index=False)['downloads'].sum()
df = df.pivot(index='date', columns='paying_customer', values='downloads').reset_index()

# filter out records of non-paying customers > than paying customers
df[df['no'] > df['yes']]
```

## Q2b: Find the rate of processed tickets for each type
Find the rate of processed tickets for each type.

### SQL
``` sql
select 
  type,
  SUM(CASE WHEN processed THEN 1
  ELSE 0 END)::float/ COUNT(complaint_id) as processed_rate
from facebook_complaints 
GROUP BY type;
```

### Python
``` python
import pandas as pd

# Group the data by type and get mean value of processed column
df = facebook_complaints.groupby('type', as_index=False)['processed'].mean()

# rename columns 
df.rename({'processed': 'processed_rate'})
df
```

## Q2c: Popularity Percentage
Find the popularity percentage for each user on Meta/Facebook. The popularity percentage is defined as the total number of friends the user has divided by the total number of users on the platform, then converted into a percentage by multiplying by 100.
Output each user along with their popularity percentage. Order records in ascending order by user id.
The 'user1' and 'user2' column are pairs of friends.

### SQL
``` sql
WITH total AS
(select 
  user1, user2
from facebook_friends l1
UNION
select 
  user2, user1
from facebook_friends l2)

SELECT
  user1,
  CAST(COUNT(user2) AS FLOAT)/(SELECT COUNT(DISTINCT user1) FROM total) * 100 AS popularity_percent
FROM total
GROUP BY user1
ORDER BY user1;
```

### Python
``` python
import pandas as pd

df = facebook_friends.copy()
df.columns = ['user2', 'user1']
df = pd.concat([facebook_friends, df])
df = df.groupby('user1', as_index=False)['user2'].count()
df['count'] = df['user2']/len(df)*100
df['count'] 
```

## Q3a: Counting Instances in Text
Find the number of times the words 'bull' and 'bear' occur in the contents. We're counting the number of times the words occur so words like 'bullish' should not be included in our count.
Output the word 'bull' and 'bear' along with the corresponding number of occurrences.

### SQL
``` sql
SELECT 
  'bull' AS word,
  COUNT(*) FROM (SELECT regexp_matches(contents, '[[:<:]]bull[[:>:]]', 'g') AS nentry
FROM google_file_store) a
UNION 
SELECT 
  'bear' AS word,
  COUNT(*) FROM (SELECT regexp_matches(contents, '[[:<:]]bear[[:>:]]', 'g') AS nentry
FROM google_file_store) b
ORDER BY word DESC
```

### Python
``` python

import pandas as pd
import re

pd.DataFrame({'word': ['bull', 'bear'],
    'netry': 
        [
            len(google_file_store['contents'].str.extractall(r'(\bbull\b)')),
            len(google_file_store['contents'].str.extractall(r'(\bbear\b)'))
        ]
})
```
## Q3b: Find matching hosts and guests in a way that they are both of the same gender and nationality
Find matching hosts and guests pairs in a way that they are both of the same gender and nationality.
Output the host id and the guest id of matched pair.

### SQL
``` sql
select
  DISTINCT host_id,
  guest_id
from airbnb_hosts h, airbnb_guests g
WHERE h.gender = g.gender AND h.nationality = g.nationality;
```

### Python
``` python

import pandas as pd

df = pd.merge(airbnb_hosts, airbnb_guests, on=['nationality', 'gender']).drop_duplicates()

df[['host_id', 'guest_id']]
```
