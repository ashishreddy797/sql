# sql
SELECT*
FROM fact_forecast_monthly;

SELECT *
FROM fact_forecast_monthly;

#What is the percentage of unique product increase in 2021 vs. 2020? 

with 
	up_2020 as
	(SELECT count(distinct product_code) as unique_products_2020
	FROM fact_forecast_monthly
	where fiscal_year LIKE "2020"),
  up_2021 as
	(SELECT count(distinct product_code) as unique_products_2021
	FROM fact_forecast_monthly
	where fiscal_year LIKE "2021")
        SELECT unique_products_2020, unique_products_2021,
        ((unique_products_2021-unique_products_2020)/unique_products_2020)*100 as percentage_chg
FROM up_2020
CROSS JOIN up_2021;

SELECT *
FROM dim_product;

#Provide a report with all the unique product counts for each segment and sort them in descending order of product counts.

SELECT segment, count(distinct product) as product_count
FROM dim_product
group by segment
ORDER BY product_count desc;

#Which segment had the most increase in unique products in 2021 vs 2020?

SELECT*
FROM fact_forecast_monthly;

SELECT*
FROM dim_product;

WITH spf as
(SELECT d.segment, f.product_code, f.fiscal_year
FROM fact_forecast_monthly f
JOIN dim_product d 
ON f.product_code=d.product_code),
pc2020 as
(SELECT count(distinct product_code) as product_count_2020
from spf
WHERE fiscal_year LIKE "2020"),
pc2021 as
(SELECT count(distinct product_code) as product_count_2021
from spf
WHERE fiscal_year LIKE "2021")
SELECT product_count_2020, product_count_2021, product_count_2021- product_count_2020 as difference
FROM pc2020
CROSS JOIN pc2021;

#Get the products that have the highest and lowest manufacturing costs. 
SELECT*
FROM dim_product;

SELECT*
FROM fact_manufacturing_cost;


(SELECT dp.product_code, dp.product, fm.manufacturing_cost
        FROM dim_product dp
		JOIN fact_manufacturing_cost fm
		ON dp.product_code = fm.product_code
order by fm.manufacturing_cost
LIMIT 1)
UNION 
(SELECT dp.product_code, dp.product, fm.manufacturing_cost
		FROM dim_product dp
		JOIN fact_manufacturing_cost fm
		ON dp.product_code = fm.product_code
order by fm.manufacturing_cost desc
LIMIT 1);

#Generate a report which contains the top 5 customers who received an average 
#high pre_invoice_discount_pct for the fiscal year 2021 and in the Indian market.

SELECT*
FROM dim_product;

SELECT*
FROM dim_customer;

SELECT* 
FROM fact_pre_invoice_deductions;

WITH cte1 as 
(SELECT ff.fiscal_year, ff.customer_code, dd.customer, fp.pre_invoice_discount_pct, dd.market
	FROM fact_forecast_monthly ff
	JOIN fact_pre_invoice_deductions fp
    ON fp.customer_code = ff.customer_code
    JOIN dim_customer dd
    ON ff.customer_code = dd.customer_code
    WHERE ff.fiscal_year LIKE 2021 AND market = "INDIA")
 SELECT fiscal_year, customer_code, customer, avg(pre_invoice_discount_pct) as avg_disc, market
 FROM cte1
 GROUP BY fiscal_year,customer_code
 ORDER BY avg_disc desc limit 5;
 
 #Get the complete report of the Gross sales amount for the customer “Atliq Exclusive” for each month.
 
 SELECT*
 FROM dim_customer
 WHERE customer = "Atliq Exclusive";
 
 SELECT*
FROM fact_gross_price;
 
SELECT*
FROM fact_sales_monthly;
 
WITH CTE1 as
(SELECT dp.product_code, fm.date, fp.gross_price, fm.sold_quantity, dc.customer
FROM dim_product dp
	JOIN fact_gross_price fp on dp.product_code = fp.product_code
    JOIN fact_sales_monthly fm on fm.product_code = fp.product_code
    JOIN dim_customer dc on dc.customer_code = fm.customer_code),
CTE2 as    
    (SELECT product_code, MONTH (date) as months, gross_price, sold_quantity, customer, gross_price*sold_quantity as gross_sales
    FROM CTE1
    WHERE customer = "Atliq Exclusive")
    SELECT product_code, months, sum(gross_sales)
    FROM CTE2
    GROUP BY product_code, months;
    
#which quarter of 2020, got the maximum total_sold_quantity?
SELECT distinct date
FROM fact_sales_monthly;

SELECT Quarter, sum(Sold_quantity) as total_sold_quantity
FROM (SELECT date,sold_quantity,
CASE 
	WHEN date >= "2019-09-01" and date < "2019-12-01" then "Q1_20"
    WHEN date >= "2019-12-01" and date < "2020-03-01" then "Q2_20"
    WHEN date >= "2020-03-01" and date < "2020-06-01" then "Q3_20"
	WHEN date >= "2020-06-01" and date < "2020-09-01" then "Q4_20"
    else "nothing"
    end as quarter
FROM fact_sales_monthly) as quarter_table
WHERE QUARTER != "nothing"
group by quarter;

#Which channel helped to bring more gross sales in the fiscal year 2021 and the percentage of contribution?


WITH CTE1 as
(SELECT fm.date, dc.customer, dc.platform, fp.gross_price* fm.sold_quantity as gross_sales
FROM dim_product dp
	JOIN fact_gross_price fp on dp.product_code = fp.product_code
    JOIN fact_sales_monthly fm on fm.product_code = fp.product_code
    JOIN dim_customer dc on dc.customer_code = fm.customer_code
WHERE date >= "2020-09-01" AND date < "2022-09-01"),

CTE2 as
(SELECT platform, round((sum(gross_sales))/1000000,2) as sales
FROM CTE1
group by platform)
SELECT platform, sales, 
round(sales*100/(SELECT sum(sales) as s FROM CTE2),2) as sales_pct
FROM CTE2
group by platform;

#Get the Top 3 products in each division that have a high total_sold_quantity in the fiscal_year 2021
SELECT distinct division
from dim_product;
SELECT*
FROM fact_sales_monthly;

WITH CTE1 as
(SELECT division, product, sum(sold_quantity) as sold_quantity
FROM dim_product dp
JOIN fact_sales_monthly fs ON dp.product_code = fs.product_code
WHERE date >= "2020-09-01" AND date < "2022-09-01"
GROUP BY division, product),

CTE2 as
(SELECT division, product, sold_quantity, 
RANK() OVER (PARTITION BY division
                    ORDER BY division, product, sold_quantity  DESC
                    ) AS price_rank
FROM CTE1)                  
SELECT division, product, sold_quantity
FROM cte2
WHERE price_rank = 1 OR price_rank = 2 OR price_rank = 3;

