# SQL_Music_Store_Analysis

create database music;
use music;
select * from album2;
-- Who is the senior most employee based on job title?

SELECT 
    *
FROM
    employee
ORDER BY levels DESC
LIMIT 1;

-- Which countries have the most Invoices?
SELECT 
    COUNT(billing_country) AS total_invoice, billing_country
FROM
    invoice
GROUP BY billing_country
ORDER BY total_invoice DESC;

-- What are top 3 values of total invoice?

select total from invoice 
order by total desc limit 3;

-- Which city has the best customers? We would like to throw a promotional Music Festival in the city we made the most money. 
-- Write a query that returns one city that has the highest sum of invoice totals. 
-- Return both the city name & sum of all invoice totals

select sum(total) as total_invoice, billing_city from invoice
group by billing_city
order by total_invoice desc limit 5;

-- Who is the best customer? The customer who has spent the most money will be declared the best customer. 
-- Write a query that returns the person who has spent the most money.

SELECT customer.customer_id, first_name, last_name, SUM(invoice.total) AS total_spending
FROM customer
JOIN invoice ON customer.customer_id = invoice.customer_id
GROUP BY customer.customer_id, first_name, last_name
ORDER BY total_spending DESC
LIMIT 5;


-- Write query to return the email, first name, last name, & Genre of all Rock Music listeners. 
-- Return your list ordered alphabetically by email starting with
select distinct email, first_name, last_name from customer
join invoice on customer.customer_id = invoice.customer_id
join invoice_line on invoice.invoice_id = invoice_line.invoice_id
join track on invoice_line.track_id = track.track_id
join genre on track.genre_id = genre.genre_id
where genre.name = 'Rock'
order by email, first_name, last_name;

-- Let's invite the artists who have written the most rock music in our dataset. 
-- Write a query that returns the Artist name and total track count of the top 10 rock bands.
select artist.artist_id,artist.name, count(artist.artist_id) as number_of_songs from track
join album2 on track.album_id = album2.album_id
join artist on album2.artist_id = artist.artist_id
join genre on track.genre_id = genre.genre_id
where genre.name like 'Rock'
group by artist.artist_id,artist.name
order by number_of_songs desc limit 10;

-- Return all the track names that have a song length longer than the average song length. 
-- Return the Name and Milliseconds for each track. Order by the song length with the longest songs listed first

select milliseconds, name from track
where milliseconds > ( select avg(milliseconds) as avg_track_length from track)
order by milliseconds desc;


/* Q1: Find how much amount spent by each customer on artists? Write a query to return customer name, artist name and total spent */

/* Steps to Solve: First, find which artist has earned the most according to the InvoiceLines. Now use this artist to find 
which customer spent the most on this artist. For this query, you will need to use the Invoice, InvoiceLine, Track, Customer, 
Album, and Artist tables. Note, this one is tricky because the Total spent in the Invoice table might not be on a single product, 
so you need to use the InvoiceLine table to find out how many of each product was purchased, and then multiply this by the price
for each artist. */

WITH best_selling_artist AS (
    SELECT
        artist.artist_id AS artist_id,
        artist.name AS artist_name,
        SUM(invoice_line.unit_price * invoice_line.quantity) AS total_sales
    FROM invoice_line
    JOIN track ON track.track_id = invoice_line.track_id
    JOIN album2 ON album2.album_id = track.album_id
    JOIN artist ON artist.artist_id = album2.artist_id
    GROUP BY artist.artist_id, artist.name
    ORDER BY total_sales DESC limit 1
)
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    bsa.artist_name,
    SUM(il.unit_price * il.quantity) AS amount_spent
FROM invoice i
JOIN customer c ON c.customer_id = i.customer_id
JOIN invoice_line il ON il.invoice_id = i.invoice_id
JOIN track t ON t.track_id = il.track_id
JOIN album2 alb ON alb.album_id = t.album_id
JOIN best_selling_artist bsa ON bsa.artist_id = alb.artist_id
GROUP BY c.customer_id, c.first_name, c.last_name, bsa.artist_name
ORDER BY amount_spent DESC;

/* Q2: We want to find out the most popular music Genre for each country. We determine the most popular genre as the genre 
with the highest amount of purchases. Write a query that returns each country along with the top Genre. For countries where 
the maximum number of purchases is shared return all Genres. */

/* Steps to Solve:  There are two parts in question- first most popular music genre and second need data at country level. */

with recursive sales_per_country as (select count(*) as purchases_per_genre, customer.country, genre.name, genre.genre_id
from invoice_line
join invoice on invoice_line.invoice_id = invoice.invoice_id
join customer on customer.customer_id = invoice.customer_id
join track on track.track_id = invoice_line.track_id
join genre on genre.genre_id = track.genre_id
group by customer.country, genre.name, genre.genre_id
order by purchases_per_genre),

max_genre_per_country as (select max(purchases_per_genre) as max_genre_number, country from sales_per_country
group by country
order by country)

select sales_per_country.* from sales_per_country
join max_genre_per_country on sales_per_country.country = max_genre_per_country.country
where sales_per_country.purchases_per_genre = max_genre_per_country.max_genre_number;

-- Q3: Write a query that determines the customer that has spent the most on music for each country. 
-- Write a query that returns the country along with the top customer and how much they spent. 
-- For countries where the top amount spent is shared, provide all customers who spent this amount.

-- Steps to Solve:  Similar to the above question. There are two parts in question- 
-- first find the most spent on music for each country and second filter the data for respective customers.


with recursive customer_with_country as ( select customer.customer_id, first_name, last_name, billing_country, sum(total) as total_spending
from invoice
join customer on customer.customer_id = invoice.customer_id
group by customer.customer_id, first_name, last_name, billing_country
order by total_spending desc),

country_max_spending as (select billing_country, max(total_spending) as max_spending from customer_with_country
group by billing_country)

select customer_with_country.billing_country,total_spending,first_name,last_name from customer_with_country
join country_max_spending on customer_with_country.billing_country = country_max_spending.billing_country
where customer_with_country.total_spending = country_max_spending.max_spending
order by 1;
