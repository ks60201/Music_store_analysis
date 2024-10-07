# Music Store Database Queries

This section contains SQL queries to analyze a music store database. Each query is designed to answer specific business questions related to customer insights, sales, and music trends.

## Q1 Query 1: Senior Most Employee by Job Title
```sql
SELECT title, last_name, first_name 
FROM employee
ORDER BY levels DESC
LIMIT 1;
```
##### This query retrieves the employee with the highest job level based on the levels column.

## Q2: Which countries have the most invoices?

```sql
SELECT COUNT(*) AS c, billing_country 
FROM invoice
GROUP BY billing_country
ORDER BY c DESC;

```
##### Finds the countries with the most invoices, ordered by the number of invoices.

## Q3: Top 3 Invoice Totals

```sql
SELECT total 
FROM invoice
ORDER BY total DESC
LIMIT 3;
```
Returns the top 3 highest invoice totals

## Q4: Which city has the best customers?

```sql
SELECT billing_city, SUM(total) AS InvoiceTotal
FROM invoice
GROUP BY billing_city
ORDER BY InvoiceTotal DESC
LIMIT 1;
```
Returns the city with the highest sum of invoice totals, identifying the city where the most revenue was generated.

## Q5: Who is the best customer?

```sql
SELECT customer.customer_id, first_name, last_name, SUM(total) AS total_spending
FROM customer
JOIN invoice ON customer.customer_id = invoice.customer_id
GROUP BY customer.customer_id
ORDER BY total_spending DESC
LIMIT 1;
```
Returns the customer who has spent the most money on invoices.

## Q6: Write a query to return the email, first name, last name, & genre of all Rock music listeners.

### Method 1

```sql
SELECT DISTINCT email, first_name, last_name
FROM customer
JOIN invoice ON customer.customer_id = invoice.customer_id
JOIN invoiceline ON invoice.invoice_id = invoiceline.invoice_id
WHERE track_id IN (
    SELECT track_id FROM track
    JOIN genre ON track.genre_id = genre.genre_id
    WHERE genre.name LIKE 'Rock'
)
ORDER BY email;
```
### Method 2
```sql
SELECT DISTINCT email AS Email, first_name AS FirstName, last_name AS LastName, genre.name AS Name
FROM customer
JOIN invoice ON invoice.customer_id = customer.customer_id
JOIN invoiceline ON invoiceline.invoice_id = invoice.invoice_id
JOIN track ON track.track_id = invoiceline.track_id
JOIN genre ON genre.genre_id = track.genre_id
WHERE genre.name LIKE 'Rock'
ORDER BY email;
```
Returns a list of customers who listen to Rock music, including their email, first name, last name, and genre, ordered by email.

## Q7: Let's invite the artists who have written the most rock music in our dataset.

```sql
SELECT artist.artist_id, artist.name, COUNT(artist.artist_id) AS number_of_songs
FROM track
JOIN album ON album.album_id = track.album_id
JOIN artist ON artist.artist_id = album.artist_id
JOIN genre ON genre.genre_id = track.genre_id
WHERE genre.name LIKE 'Rock'
GROUP BY artist.artist_id
ORDER BY number_of_songs DESC
LIMIT 10;
```
Returns the top 10 rock artists based on the number of rock tracks they have written, ordered by the number of songs.

## Q8: Return all the track names that have a song length longer than the average song length.

```sql
SELECT name, miliseconds
FROM track
WHERE miliseconds > (
    SELECT AVG(miliseconds) AS avg_track_length
    FROM track
)
ORDER BY miliseconds DESC;
```
Returns all track names with a song length longer than the average, ordered by the song length in descending order.

## Q9: How much amount was spent by each customer on artists?

```sql
WITH best_selling_artist AS (
    SELECT artist.artist_id AS artist_id, artist.name AS artist_name, 
           SUM(invoice_line.unit_price * invoice_line.quantity) AS total_sales
    FROM invoice_line
    JOIN track ON track.track_id = invoice_line.track_id
    JOIN album ON album.album_id = track.album_id
    JOIN artist ON artist.artist_id = album.artist_id
    GROUP BY 1
    ORDER BY 3 DESC
    LIMIT 1
)
SELECT c.customer_id, c.first_name, c.last_name, bsa.artist_name, 
       SUM(il.unit_price * il.quantity) AS amount_spent
FROM invoice i
JOIN customer c ON c.customer_id = i.customer_id
JOIN invoice_line il ON il.invoice_id = i.invoice_id
JOIN track t ON t.track_id = il.track_id
JOIN album alb ON alb.album_id = t.album_id
JOIN best_selling_artist bsa ON bsa.artist_id = alb.artist_id
GROUP BY 1, 2, 3, 4
ORDER BY 5 DESC;
```
Returns the total amount spent by each customer on the best-selling artist in the dataset.

## Q10: What is the most popular music genre for each country?

### Method 1: Using CTE

```sql
WITH popular_genre AS (
    SELECT COUNT(invoice_line.quantity) AS purchases, customer.country, genre.name, genre.genre_id, 
           ROW_NUMBER() OVER(PARTITION BY customer.country ORDER BY COUNT(invoice_line.quantity) DESC) AS RowNo 
    FROM invoice_line 
    JOIN invoice ON invoice.invoice_id = invoice_line.invoice_id
    JOIN customer ON customer.customer_id = invoice.customer_id
    JOIN track ON track.track_id = invoice_line.track_id
    JOIN genre ON genre.genre_id = track.genre_id
    GROUP BY 2, 3, 4
    ORDER BY 2 ASC, 1 DESC
)
SELECT * FROM popular_genre WHERE RowNo <= 1;
```
### Method 2: Using Recursive
```sql
WITH RECURSIVE sales_per_country AS (
    SELECT COUNT(*) AS purchases_per_genre, customer.country, genre.name, genre.genre_id
    FROM invoice_line
    JOIN invoice ON invoice.invoice_id = invoice_line.invoice_id
    JOIN customer ON customer.customer_id = invoice.customer_id
    JOIN track ON track.track_id = invoice_line.track_id
    JOIN genre ON genre.genre_id = track.genre_id
    GROUP BY 2, 3, 4
    ORDER BY 2
),
max_genre_per_country AS (
    SELECT MAX(purchases_per_genre) AS max_genre_number, country
    FROM sales_per_country
    GROUP BY 2
    ORDER BY 2
)
SELECT sales_per_country.* 
FROM sales_per_country
JOIN max_genre_per_country ON sales_per_country.country = max_genre_per_country.country
WHERE sales_per_country.purchases_per_genre = max_genre_per_country.max_genre_number;
```
Returns the most popular genre for each country, determined by the highest number of purchases. For countries with a tie, all top genres are returned.

