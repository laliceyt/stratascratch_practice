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