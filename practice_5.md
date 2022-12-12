## Q1a. Book Sales
Calculate the total revenue made per book. Output the book ID and total sales per book. 
In case there is a book that has never been sold, include it in your output with a value of 0.

### SQL
```sql 
SELECT
  b.book_id,
  IFNULL(quantity * unit_price, 0) AS total_sales 
FROM
  amazon_books b 
  LEFT JOIN
    (
      SELECT
        book_id,
        SUM(quantity) AS quantity 
      FROM
        amazon_books_order_details 
      GROUP BY
        book_id 
    )
    agg_order 
    ON b.book_id = agg_order.book_id
```

### Python
```python
# Import your libraries
import pandas as pd

# Start writing code
amazon_books.head()

# Get aggregated quantity
agg_orders = amazon_books_order_details.groupby('book_id', as_index=False)['quantity'].sum()

# Left join the two df
books_df = amazon_books.merge(agg_orders, how='left')

books_df = books_df[['book_id', 'unit_price', 'quantity']]

# Create total sales col
books_df['total_sales'] = books_df['quantity'] * books_df['unit_price']

# Fill rows without values with 0
books_df['total_sales'] = books_df['total_sales'].fillna(0)

books_df[['book_id', 'total_sales']]
```

## Q1b. Post Likes
You are given a list of posts of a Facebook user. Find the average number of likes.

### SQL
```sql 
SELECT
  AVG(no_of_likes) AS AVG 
FROM
  fb_posts;
```

### Python
```python
fb_posts['no_of_likes'].mean()
```
