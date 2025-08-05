# Consumer-Goods-Ad-hoc-insights

# Request 1.0
--  Provide the list of markets in which customer "Atliq Exclusive" operates its business in the APAC region.

SELECT
   *  FROM dim_customer 
where
    customer = "Atliq Exclusive" and region = "APAC";

#Request 2.0  
 -- Provide a report with all the unique product counts for each segment and  sort them in descending order of product counts. 
 
  SELECT segment, count(product_code) as product_count  FROM gdb023.dim_product group by segment order by product_count desc;
  
  # Request 3.0
  -- What is the percentage of unique product increase in 2021 vs. 2020?
  
with product_yearly_sales as
(SELECT 
    COUNT(DISTINCT CASE WHEN fiscal_year = 2020 THEN product_code END) AS unique_products_2020,
    COUNT(DISTINCT CASE WHEN fiscal_year = 2021 THEN product_code END) AS unique_products_2021
FROM 
    fact_sales_monthly
WHERE 
    fiscal_year IN (2020, 2021))
    select unique_products_2020, unique_products_2021, round((unique_products_2021- unique_products_2020)*100/unique_products_2020,2) as pct_chg from product_yearly_sales;
    
     
# Request 4.0
 -- Provide a report with all the unique product counts for each segment and sort them in descending order of product counts   
    
  with table_1 as
(select p.segment, p.product_code, s.fiscal_year 
from dim_product p 
join fact_sales_monthly s 
on p.product_code=s.product_code),

table_2 AS (
    SELECT segment,
        COUNT(DISTINCT CASE WHEN fiscal_year = 2020 THEN product_code END) AS product_count_2020,
        COUNT(DISTINCT CASE WHEN fiscal_year = 2021 THEN product_code END) AS product_count_2021
        from table_1
        group by segment)

SELECT segment, product_count_2020,product_count_2021, (product_count_2021 -product_count_2020) as difference
 FROM table_2
 group by segment;
 
#Request 5.0
 --  Get the products that have the highest and lowest manufacturing costs.
 select * from
(SELECT
m.product_code, p.product, m.manufacturing_cost
 FROM fact_manufacturing_cost m
 join dim_product p
 using(product_code)
 order by m.manufacturing_cost desc
 limit 1) as highest
 union
 select * from 
 (SELECT
m.product_code, p.product, m.manufacturing_cost
 FROM fact_manufacturing_cost m
 join dim_product p
 using(product_code)
 order by m.manufacturing_cost asc
 limit 1) as lowest;
 
 #Request 6.0
 -- Generate a report which contains the top 5 customers who received an average high pre_invoice_discount_pct for the fiscal year 2021 and in the Indian market. 
 
  SELECT
 i.customer_code,
 c.customer,
 ROUND(AVG(pre_invoice_discount_pct) * 100, 2) AS avg_dct_pct
FROM gdb023.fact_pre_invoice_deductions i
JOIN dim_customer c USING (customer_code)
JOIN fact_sales_monthly s USING (customer_code)
WHERE c.market = "india" AND s.fiscal_year = 2021
GROUP BY i.customer_code, c.customer
ORDER BY avg_dct_pct DESC
LIMIT 5;

# Request 7.0
-- Get the complete report of the Gross sales amount for the customer “Atliq Exclusive” for each month.
with table1 as 
(select monthname(date) as month,m.sold_quantity,m.fiscal_year,g.gross_price
from fact_sales_monthly m
join fact_gross_price g
using (product_code))
select  month, fiscal_year , round(sum(gross_price*sold_quantity),2) as gross_amount from table1
group by month ,fiscal_year;

# Request 8.0
-- In which quarter of 2020, got the maximum total_sold_quantity?
with table1 as(
SELECT fiscal_year, sold_quantity,
  CASE 
    WHEN MONTH(date) IN (9,10,11) THEN 'Q1'
    WHEN MONTH(date) IN (12,1,2) THEN 'Q2'
    WHEN MONTH(date) IN (3,4,5) THEN 'Q3'
    WHEN MONTH(date) IN (6,7,8) THEN 'Q4'
  END AS Quarter
  from fact_sales_monthly)
  select Quarter, sum(sold_quantity) as total_sold_qty
  from table1
  where fiscal_year = 2020
  group by Quarter
  order by total_sold_qty desc;
  
  # Request 9.0
 -- Which channel helped to bring more gross sales in the fiscal year 2021 and the percentage of contribution?
   with table1 as 
(SELECT s.customer_code, s.sold_quantity, g.gross_price 
FROM fact_sales_monthly s
join fact_gross_price g
using (product_code)
where s.fiscal_year = 2021),
 table2 as
(select c.channel, round(sum((sold_quantity*gross_price)/1000000),2) as gross_amount_mln
 from table1 t
join dim_customer c
using (customer_code) 
group by channel)
select  channel, gross_amount_mln, round((gross_amount_mln*100/(sum(gross_amount_mln) over())),2) as gross_amount_pct
from table2
order by gross_amount_mln desc;


# Request 10.0 
 -- Get the Top 3 products in each division that have a high total_sold_quantity in the fiscal_year 2021? 
 
 WITH table1 AS (
  SELECT 
    p.division,
    p.product_code,
    p.product,
    SUM(sold_quantity) AS total_sold_qty
  FROM gdb023.dim_product p
  JOIN fact_sales_monthly f USING(product_code)
  GROUP BY p.division, p.product_code, p.product
),
ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY division ORDER BY total_sold_qty DESC) AS rnk_order
  FROM table1
)
SELECT division, product_code, product, total_sold_qty, rnk_order
FROM ranked
WHERE rnk_order <= 3;
