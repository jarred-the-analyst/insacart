


https://github.com/jarred-the-analyst/insacart/assets/136924918/7a748c59-e2d2-4b91-9f3b-2c580cb1160a



https://github.com/jarred-the-analyst/insacart/assets/136924918/5d68cab6-0c7b-4086-a638-e659c0922bce



# inventory of products
 ```mysql
select 
	department,
	count(d.department_id) as t_d,
	cast((count(d.department_id)/cast(sum(count(d.department_id)) over () as numeric))*100 as decimal(10,2))
from 
	instacart_p.dbo.products as p left join instacart_p.dbo.departments as d on p.department_id=d.department_id
	left join instacart_p.dbo.aisle as a on p.aisle_id=a.aisle_id

group by
	d.department
order by t_d desc
	;
```
# total sales for each department
 ```mysql
with tmp as 
	(
		select 
			*
		from 
			instacart_p.dbo.order_products_prior 
		union
		select 
			*
		from 
			instacart_p.dbo.order_products_train
	)

		select
		department,
		count(*) as t_prod_in_dept,
		cast(round((count(*)/cast(sum(count(*)) over () as numeric)*100),2) as float) precentage_o_sale,
		rank() over (order by count(*) desc) ranking

		from instacart_p.dbo.products p join instacart_p.dbo.departments d on p.department_id=d.department_id
			 right join tmp on p.product_id=tmp.product_id

		group by 
			department
```
-- quartiles of data

with tmp as 
	(
		select 
			*
		from 
			instacart_p.dbo.order_products_prior 
		union
		select 
			*
		from 
			instacart_p.dbo.order_products_train
	), tmp2 as 
	(select
			d.department as dept,
			order_id


		from instacart_p.dbo.products p join instacart_p.dbo.departments d on p.department_id=d.department_id
		join tmp on p.product_id=tmp.product_id
)
, tmp3 as(select
			order_id,
			count(*) as how_many_items_in_basket

		from instacart_p.dbo.products p join instacart_p.dbo.departments d on p.department_id=d.department_id
			  join tmp on p.product_id=tmp.product_id

		group by 
			order_id),
tmp4 as (select
	distinct dept,
	how_many_items_in_basket,
	tmp2.order_id
from tmp2 join tmp3 on tmp2.order_id=tmp3.order_id)
, tmp5 as (select 
dept,
how_many_items_in_basket,
ntile(100) over (partition by dept order by how_many_items_in_basket) as per
from tmp4)
select
	min(dept),
	min(how_many_items_in_basket)
from tmp5
group by dept, per

-- creating table 

DROP TABLE #order_dept_produ_table;
CREATE TABLE #order_dept_produ_table (
  product_id INT,
  product_name VARCHAR(255),
  aisle_id INT,
  department_id INT,
  department VARCHAR(100),
  order_id INT,
  add_to_cart_order INT,
  reordered INT
);

with tmp as 
	(
		select 
			*
		from 
			instacart_p.dbo.order_products_prior 
		union
		select 
			*
		from 
			instacart_p.dbo.order_products_train
	)
INSERT INTO #order_dept_produ_table (product_id, product_name, aisle_id, department_id, department, order_id, add_to_cart_order, reordered)
select
	p.product_id,product_name,aisle_id,d.department_id,department,order_id,add_to_cart_order,reordered
from instacart_p.dbo.products p join instacart_p.dbo.departments d on p.department_id=d.department_id
right join tmp on p.product_id=tmp.product_id;

-- amount of user purcahse in giving dept 
with tmp as
(	select 
		count(distinct user_id) as total_users
from #order_dept_produ_table odp join instacart_p.dbo.orders o on odp.order_id=o.order_id

)
, tmp2 as (select 
	department,
	count(distinct user_id) amount_purchase_per_d
from #order_dept_produ_table odp join instacart_p.dbo.orders o on odp.order_id=o.order_id 
group by
	department)
select
	department,
	amount_purchase_per_d,
	cast((amount_purchase_per_d/cast(total_users as numeric))*100 as decimal(10,3)) as amount_o_user_who_purcase_d
from tmp2 cross join tmp
order by amount_o_user_who_purcase_d desc

-- average reorder time
select 	
	department,
	cast(avg(days_since_prior_order)as decimal(10,2)) avg_time_for_reorder,
	rank() over (order by avg(days_since_prior_order)) rank_avg_time_for_reorder
from #order_dept_produ_table odp join instacart_p.dbo.orders o on odp.order_id=o.order_id
where reordered=1
group by 
	department 
-- likeihood_of_reorder
with tmp as
(	-- manipulating data so i can preform functions on users who have reorder items in specific deparments
		select 
			user_id,
			department,
			sum(reordered) how_many_times_reorder ,
			case when sum(reordered) > 0  then 1 else 0 end as dept_reorder

		from #order_dept_produ_table odp join instacart_p.dbo.orders o on odp.order_id=o.order_id
		group by
			user_id,department
)
		-- finding likelihood of reorder by department [favorable_events/total_events], finding total reordes in dept and ranking the amount of reorders
		select 
			sum(dept_reorder) users_who_reoderd,
			cast(sum(dept_reorder)/cast(count(dept_reorder) as decimal(10,3))*100 as decimal(10,3)) as liklie_of_reordring, 
			department,
			rank() over (order by sum(dept_reorder) desc) rank_of_user_who_reorder
		from tmp
		group by 
			department

-- reorder ratio

select 
	department,
	sum(reordered) num_reorders,
	cast(100*(sum(reordered)/cast(count(*) as numeric))as decimal (10,2)) reorder_ratio
from #order_dept_produ_table odp left join instacart_p.dbo.orders o on odp.order_id=o.order_id
group by 
	department
order by reorder_ratio desc

-- median amount items in basket
with tmp as 
	(
		select 
			*
		from 
			instacart_p.dbo.order_products_prior 
		union
		select 
			*
		from 
			instacart_p.dbo.order_products_train
	)  , tmp2 as

 (
		select
			order_id,
			count(*) as how_many_items_in_basket,
			ntile(100) over (order by count(*)) as percentiles

		from instacart_p.dbo.products p join instacart_p.dbo.departments d on p.department_id=d.department_id
			  join tmp on p.product_id=tmp.product_id

		group by 
			order_id
), tmp3 as
(
		select
			order_id,
			how_many_items_in_basket,
			ntile(100) over (order by how_many_items_in_basket) as percentile
		from 
			tmp2
		where order_id in (select order_id from instacart_p.dbo.products p join instacart_p.dbo.departments d on p.department_id=d.department_id
			 right join tmp on p.product_id=tmp.product_id where department='personal care')

)
select 
	min(tmp2.how_many_items_in_basket) median_purchase_total,
	min(tmp3.how_many_items_in_basket) median_purchase_personal_c
from 
	tmp2 join tmp3 on tmp2.percentiles=tmp3.percentile
where percentiles >=50 and percentile >= 50;


the meadian amount of items in the basket is much higher in the personal care department. potential upsale.
use median because of outliers like the 127 items in cart. avarage would be skwed 

median_purchase_total	median_purchase_personal_c

8				12

