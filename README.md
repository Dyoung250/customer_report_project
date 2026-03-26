## DATA ANALYTICS:
## FIRST STAGE	  	----> CHANGE OVER TIME:

#### Below we will look at different method of analyzing the change over time based on different date function that we are going to group and order our data as by month, yearly, month and year which we will use datetrunc, and then the date format function with the application of aggregate functions. The change over time will guide us to analyzed our product in a yearly or monthly point of view.


```sql
select
    count(distinct order_id) as total_orders,
	year(order_date) as yearly_order_date,
	month(order_date) as monthly_order_date,
	sum(revenue) as total_revenue,
	sum(quantity) as total_quantity,
	count(distinct customer_id) as total_custmers,
	sum(cost) as total_cost,
	sum(profit) as total_profit
from sales
where order_date  is not null
group by order_date
order by monthly_order_date;
 ``` 
```sql	
	select
	datetrunc(year, order_date) as yearly_order,
    count(distinct order_id) as total_orders,
	sum(revenue) as total_revenue,
	sum(quantity) as total_quantity,
	count(distinct customer_id) as total_custmers,
	sum(cost) as total_cost,
	sum(profit) as total_profit
from sales
where order_date  is not null
group by datetrunc(year, order_date)
order by datetrunc(year,order_date) ;
```
```sql 
  select
	year(order_date) as yearly_order,
    count(distinct order_id) as total_orders,
	sum(revenue) as total_revenue,
	sum(quantity) as total_quantity,
	count(distinct customer_id) as total_custmers,
	sum(cost) as total_cost,
	sum(profit) as total_profit
from sales
where year(order_date)  is not null
group by year( order_date);
```
 ```sql
  select
	month(order_date) as monthly_order,
    count(distinct order_id) as total_orders,
	sum(revenue) as total_revenue,
	sum(quantity) as total_quantity,
	count(distinct customer_id) as total_custmers,
	sum(cost) as total_cost,
	sum(profit) as total_profit
from sales
where order_date  is not null
group by month( order_date)
order by month( order_date) asc ;
```
```sql
select
	format(order_date, 'yyyy, MMM') as monthly_order,
    count(distinct order_id) as total_orders,
	sum(revenue) as total_revenue,
	sum(quantity) as total_quantity,
	count(distinct customer_id) as total_custmers,
	sum(cost) as total_cost,
	sum(profit) as total_profit
from sales
where order_date  is not null
group by format(order_date, 'yyyy, MMM')
order by format(order_date, 'yyyy, MMM') asc ;
```



## ADVANCED ANALYTICS PROJECT:
## SECOND ANALYSIS ---> CUMULATIVE ANALYSIS :

### Aggregate the data progressively overtime.
#### This helps us to understand whether our business is growing or declining.In this type of analysis, we will use the aggregate window functions:

#### Calculate the total revenue,total cost and total profit for each month,the running totals over-time, and their moving averages 3-day.


#### --- Here we calculates the running totals on monthly bases.
```sql
with Running_totals as (
select
	datetrunc(month,order_date) as order_date,
	sum(revenue) as total_revenue,
	sum(cost) as total_cost,
	sum(profit) as total_profit
from sales
where order_date is not null
group by datetrunc(month, order_date)
),
--- calculating running total for total revenue, total cost and total profit
 moving_average as (
select
	order_date,
	total_revenue,
	total_cost,
	total_profit,
	sum(total_revenue) over (order by order_date) as running_total_revenue,
	sum(total_cost) over (order by order_date) as running_total_cost,
	sum(total_profit) over (order by order_date) as running_total_profit
from Running_totals
group by
		order_date,
	total_revenue,
	total_cost,
	total_profit
)
--- calculating moving averages for total revenue, total cost and total profit
select
	order_date,
	total_revenue,
	total_cost,
	total_profit,
	running_total_revenue,
	running_total_cost,
	running_total_profit,
	round(avg(total_revenue) over (order by order_date rows between 2 preceding and current row),0) as revenue_moving_average,
	round(avg(total_cost) over (order by order_date rows between 2 preceding and current row),0) as cost_moving_average,
	round(avg(total_profit) over (order by order_date rows between 2 preceding and current row),0) as profit_moving_average
from moving_average;
```



## THIRD  STAGE ANALYSIS  ---> PERFORMANCE ANALYSIS:

### Here we compare the current value to a target value.
#### It helps us to measure success and compare performance in the sense that we 
#### subtract the current[measure] - target[measure]
#### for example: current revenue - average revenue
####              current year revenue - previous year revenue.  ie --> yoy Analysis. 
####              current revenue - lowest revenue 
####              current year cost - previous year cost
####              current year profit - provius year cost
####              current year quantity - previous year quantity


#### task:  analyze the yearly performance of the products by comparing each product's Revenue, Cost, and Profit toboth its average Revenue, Cost, quantity and Profit performance and the prevous year's revenue.
```sql
with performance_analysis as (
select
	year(s.order_date) as order_year,
	p.product_name,
	p.brand,
	p.category,
	sum(s.revenue) as current_revenue,
	sum(s.cost) as current_cost,
	sum(s.quantity) as current_quantity,
	sum(s.profit) as current_profit
from sales as s
left join products as p
on s.product_id = p.product_id
where order_date is not null
	and product_name is not null
	and brand is not null
	and category is not null
group by year(s.order_date), 
		 p.product_name,
	     p.brand,
	     p.category
)
select
         order_year,
	     product_name,
	     brand,
	     category,
		 current_revenue,
		 current_cost,
		 current_quantity,
		 current_profit,
		 round(AVG(current_revenue) over (partition by product_name),0) as avg_revenue,
		 round(current_revenue - AVG(current_revenue) over (partition by product_name),0) as revenue_diff_avg,
		 case when round(current_revenue - AVG(current_revenue) over (partition by product_name),0) > 0 then 'above average'
			  when round(current_revenue - AVG(current_revenue) over (partition by product_name),0) < 0 then 'below average'
			  else 'average'
			  end as revenue_avg_change,
		 round(avg(current_cost) over (partition by product_name),0) as avg_cost,
		 round(current_cost - avg(current_cost) over (partition by product_name),0) as cost_diff_avg,
		 case when round(current_cost - avg(current_cost) over (partition by product_name),0) > 0 then 'above average'
		      when round(current_cost - avg(current_cost) over (partition by product_name),0) < 0 then 'below average'
			  else 'average'
			  end as cost_avg_change,
		 round(avg(current_quantity) over (partition by product_name),0) as avg_quantity,
		 round(current_quantity - avg(current_quantity) over (partition by product_name),0) as quantity_diff_avg,
		 case when round(current_quantity - avg(current_quantity) over (partition by product_name),0) > 0 then 'above average'
			  when round(current_quantity - avg(current_quantity) over (partition by product_name),0) < 0 then 'below average'
			  else 'average'
			  end as quantity_avg_change,
		 round(avg(current_profit) over (partition by product_name),0) as avg_profit,
		 round(current_profit - avg(current_profit) over (partition by product_name),0) as profit_diff_avg,
		 case when round(current_profit - avg(current_profit) over (partition by product_name),0) > 0 then 'above average'
			  when round(current_profit - avg(current_profit) over (partition by product_name),0) < 0 then 'below average'
			  else 'average'
			  end as profit_avg_change
from performance_analysis;
```	
## Yearly revenue performance: On this part we analyzed the yearly revenue performance.
```sql
	with revenue_performance as (
select
	datetrunc(year,s.order_date) as order_year,
	p.product_name,
	p.brand,
	p.category,
	sum(s.revenue) as current_revenue	
from sales as s
left join products as p
on s.product_id = p.product_id
where order_date is not null
	and product_name is not null
	and brand is not null
	and category is not null
group by datetrunc(year,s.order_date), 
		 p.product_name,
	     p.brand,
	     p.category
)
select
         order_year,
	     product_name,
	     brand,
	     category,
		 current_revenue,
		 round(AVG(current_revenue) over (partition by product_name),2) as avg_revenue,
		 round(current_revenue - AVG(current_revenue) over (partition by product_name),2) as revenue_diff_avg,
		 case when round(current_revenue - AVG(current_revenue) over (partition by product_name),2) > 0 then 'above average'
			  when round(current_revenue - AVG(current_revenue) over (partition by product_name),2) < 0 then 'below average'
			  else 'average'
			  end as revenue_avg_change,	
		 lag(current_revenue) over (partition by product_name order by order_year) as previous_year_sales,
		 current_revenue - lag(current_revenue) over (partition by product_name order by order_year) as diff_py_sales,
		 case when current_revenue - lag(current_revenue) over (partition by product_name order by order_year) > 0 then 'increase'
			  when current_revenue - lag(current_revenue) over (partition by product_name order by order_year) < 0 then 'decrease'
			  else 'no change'
			  end as py_change
from revenue_performance;
	```			

--- Monthly revenue performance analysis.
```sql
with revenue_performance as (
select
	datetrunc(month,s.order_date) as order_month,
	p.product_name,
	p.brand,
	p.category,
	sum(s.revenue) as current_revenue	
from sales as s
left join products as p
on s.product_id = p.product_id
where order_date is not null
	and product_name is not null
	and brand is not null
	and category is not null
group by datetrunc(month,s.order_date), 
		 p.product_name,
	     p.brand,
	     p.category
)
select
         order_month,
	     product_name,
	     brand,
	     category,
		 current_revenue,
		 round(AVG(current_revenue) over (partition by product_name),2) as avg_revenue,
		 round(current_revenue - AVG(current_revenue) over (partition by product_name),2) as revenue_diff_avg,
		 case when round(current_revenue - AVG(current_revenue) over (partition by product_name),2) > 0 then 'above average'
			  when round(current_revenue - AVG(current_revenue) over (partition by product_name),2) < 0 then 'below average'
			  else 'average'
			  end as revenue_avg_change,	
		 lag(current_revenue) over (partition by product_name order by order_month) as previous_year_sales,
		 current_revenue - lag(current_revenue) over (partition by product_name order by order_month) as diff_py_sales,
		 case when current_revenue - lag(current_revenue) over (partition by product_name order by order_month) > 0 then 'increase'
			  when current_revenue - lag(current_revenue) over (partition by product_name order by order_month) < 0 then 'decrease'
			  else 'no change'
			  end as py_change
from revenue_performance;
```			  


#### Yearly cost performance analysis:
```sql
with cost_performance as (
select
	datetrunc(year,s.order_date) as order_year,
	p.product_name,
	p.brand,
	p.category,
	sum(s.cost) as current_cost	
from sales as s
left join products as p
on s.product_id = p.product_id
where order_date is not null
	and product_name is not null
	and brand is not null
	and category is not null
group by datetrunc(year,s.order_date), 
		 p.product_name,
	     p.brand,
	     p.category
)
select
         order_year,
	     product_name,
	     brand,
	     category,
		 current_cost,
		 round(AVG(current_cost) over (partition by product_name),2) as avg_cost,
		 round(current_cost - AVG(current_cost) over (partition by product_name),2) as cost_diff_avg,
		 case when round(current_cost - AVG(current_cost) over (partition by product_name),2) > 0 then 'above average'
			  when round(current_cost - AVG(current_cost) over (partition by product_name),2) < 0 then 'below average'
			  else 'average'
			  end as cost_avg_change,	
		 lag(current_cost) over (partition by product_name order by order_year) as previous_year_cost,
		 current_cost - lag(current_cost) over (partition by product_name order by order_year) as diff_py_cost,
		 case when current_cost - lag(current_cost) over (partition by product_name order by order_year) > 0 then 'increase'
			  when current_cost - lag(current_cost) over (partition by product_name order by order_year) < 0 then 'decrease'
			  else 'no change'
			  end as py_change
from cost_performance;
```	

#### Yearly quantity performance analysis: here we analyzed the quantity of prouducts that has been sold. 
```sql
with quantity_performance as (
select
	datetrunc(year,s.order_date) as order_year,
	p.product_name,
	p.brand,
	p.category,
	sum(s.quantity) as current_quantity	
from sales as s
left join products as p
on s.product_id = p.product_id
where order_date is not null
	and product_name is not null
	and brand is not null
	and category is not null
group by datetrunc(year,s.order_date), 
		 p.product_name,
	     p.brand,
	     p.category

)
select
         order_year,
	     product_name,
	     brand,
	     category,
		 current_quantity,
		 round(AVG(current_quantity) over (partition by product_name),2) as avg_quantity,
		 round(current_quantity - AVG(current_quantity) over (partition by product_name),2) as quantity_diff_avg,
		 case when round(current_quantity - AVG(current_quantity) over (partition by product_name),2) > 0 then 'above average'
			  when round(current_quantity - AVG(current_quantity) over (partition by product_name),2) < 0 then 'below average'
			  else 'average'
			  end as quantity_avg_change,	
		 lag(current_quantity) over (partition by product_name order by order_year) as previous_year_quantity,
		 current_quantity - lag(current_quantity) over (partition by product_name order by order_year) as diff_py_quantity,
		 case when current_quantity - lag(current_quantity) over (partition by product_name order by order_year) > 0 then 'increase'
			  when current_quantity - lag(current_quantity) over (partition by product_name order by order_year) < 0 then 'decrease'
			  else 'no change'
			  end as py_change
from quantity_performance;
```	
	

#### Yearly profit performance: Here we analyzed the performance based on profit on a yearly base.
```sql
with profit_performance as (
select
	datetrunc(year,s.order_date) as order_year,
	p.product_name,
	p.brand,
	p.category,
	sum(s.profit) as current_profit	
from sales as s
left join products as p
on s.product_id = p.product_id
where order_date is not null
	and product_name is not null
	and brand is not null
	and category is not null
group by datetrunc(year,s.order_date), 
		 p.product_name,
	     p.brand,
	     p.category
)
select
         order_year,
	     product_name,
	     brand,
	     category,
		 current_profit,
		 round(AVG(current_profit) over (partition by product_name),2) as avg_profit,
		 round(current_profit - AVG(current_profit) over (partition by product_name),2) as profit_diff_avg,
		 case when round(current_profit - AVG(current_profit) over (partition by product_name),2) > 0 then 'above average'
			  when round(current_profit - AVG(current_profit) over (partition by product_name),2) < 0 then 'below average'
			  else 'average'
			  end as profit_avg_change,	
		 lag(current_profit) over (partition by product_name order by order_year) as previous_year_profit,
		 current_profit - lag(current_profit) over (partition by product_name order by order_year) as diff_py_profit,
		 case when current_profit - lag(current_profit) over (partition by product_name order by order_year) > 0 then 'increase'
			  when current_profit - lag(current_profit) over (partition by product_name order by order_year) < 0 then 'decrease'
			  else 'no change'
			  end as py_change
from profit_performance;
```	


## LAST STAGE ---> building customer report:
###  customer report:
###  purporse:
#### This report consolidates fields such as names, ages, and transaction details.
### Highlights:
		1. Gathers essential fields such as customer_id, age, gender and transaction details.
		2. Segments customers into categories (vip, regular, new_customer) and age groups.
		3. Aggregates customer-level metrics:
				- total orders
				- total revenue
				- total quantity purchased
				- total products 
				- lifespan (in months)
		4. Calculates valuable kpis:
				- recency (months since last order )
				- latency (months since first_order)
				- average order value 
				- average monthly spend.
	

##   Customers' report Analysis:
#### Customer Aggregatons:  Here summerizes key metrics at the customer level.
```sql
with customer_report as (
select
	c.join_date,
	c.age,
	c.gender,
	datetrunc(year, s.order_date) as order_year,
	min(s.order_date) as first_order_date,
	max(s.order_date) as last_order_date,
	datediff(month, min(s.order_date), max(s.order_date)) as lifespan_in_months,
	count(s.order_id) as total_orders,
	count(s.product_id) as total_product,
	count(distinct s.customer_id) as total_customers,
	sum(s.unit_price) as total_price,
	sum(s.quantity) as total_quantity,
	sum(s.revenue) as total_spending
from sales as s
left join customers as c
on s.customer_id = c.customer_id
LEFT join stores as t
on s.store_id = t.store_id
where order_date is not null
	and order_id is not null
	and join_date is not null
group by 
		 c.join_date,
		 c.age,
		 c.gender,
		 datetrunc(year, s.order_date)
)
select 
	join_date,
	age,
	case when age < 20 then 'under 20'
		 when age between 20 and 29 then '20 - 29'
		 when age between 30 and 39 then '30 - 39'
		 when age between 40 and 49 then '40 - 49'
		 else '50 and above'
		 end as age_group,
	gender,
	order_year,
	first_order_date,
	datediff(month, first_order_date, getdate()) as latency,
	last_order_date,
	datediff(month, last_order_date, getdate()) as recency,
	lifespan_in_months,
	case when lifespan_in_months >= 10 and total_spending > 51000 then 'vip customer'
		 when lifespan_in_months >= 10 and total_spending <= 50000 then 'regular customer'
		 else 'new customer'
		 end as customer_segment,
	total_orders,
	total_product,
	total_customers,
	total_price,
	round(avg(total_price) over (order by order_year),0) as avg_total_price,
	total_quantity,
	AVG(total_quantity) over (order by order_year) as avg_quantity,
	total_quantity - AVG(total_quantity) over (order by order_year) as diff_avg_quantity,
case when total_quantity - AVG(total_quantity) over (order by order_year) > 0 then 'above average'
	 when total_quantity - AVG(total_quantity) over (order by order_year) < 0 then 'below average'
	 else 'average'
	 end as avg_quantity_change,
	 total_quantity - LAG(total_quantity) over (order by order_year) as previous_year_qty,
case when total_quantity - LAG(total_quantity) over (order by order_year) > 0 then 'increase'
	 when total_quantity - LAG(total_quantity) over (order by order_year) < 0 then 'decrease'
	 else 'no change'
	 end as py_change,
	total_spending,
--- compute average order value (AOV)
	case when      total_spending = 0 then 0
	     else      round(total_spending / total_orders,0)
		 end as average_order_value,
---- compute average monthly spend
	case when lifespan_in_months = 0 then total_spending
		 else round(total_spending / lifespan_in_months,0)
		 end as average_monthly_spend
from customer_report;
 ```

 
