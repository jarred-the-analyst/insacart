

https://github.com/jarred-the-analyst/insacart/assets/136924918/7eb3f241-056b-46a8-9231-7b3f22c0c980



https://github.com/jarred-the-analyst/insacart/assets/136924918/c6cf07cd-0127-4235-90c4-4992b65ecb40


https://github.com/jarred-the-analyst/insacart/assets/136924918/768591b2-4382-442c-a9d0-f8549e845888







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

https://github.com/jarred-the-analyst/insacart/assets/136924918/61d5444e-1605-46d4-8457-81711a0a4816



https://github.com/jarred-the-analyst/insacart/assets/136924918/bdcce613-371f-46d2-9c3e-bb63bcdd3d27


with tmp as 
	(
		select 
			*

https://github.com/jarred-the-analyst/insacart/assets/136924918/1f6bb14f-5aeb-4b02-b554-9514502ca06d


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
# quartiles of data

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
```

# creating table 
 ```mysql
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
```
# amount of user purcahse in giving dept
 ```mysql
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
```

# average reorder time
 ```mysql
select 	
	department,
	cast(avg(days_since_prior_order)as decimal(10,2)) avg_time_for_reorder,
	rank() over (order by avg(days_since_prior_order)) rank_avg_time_for_reorder
from #order_dept_produ_table odp join instacart_p.dbo.orders o on odp.order_id=o.order_id
where reordered=1
group by 
	department
```
# likeihood_of_reorder
 ```mysql
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
```

# reorder ratio
 ```mysql
select 
	department,
	sum(reordered) num_reorders,
	cast(100*(sum(reordered)/cast(count(*) as numeric))as decimal (10,2)) reorder_ratio
from #order_dept_produ_table odp left join instacart_p.dbo.orders o on odp.order_id=o.order_id
group by 
	department
order by reorder_ratio desc
```

# median amount items in basket when personal care was purchase first
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
	dept as cat,
	min(how_many_items_in_basket) per
from tmp5
where per >= 50
group by dept
order by 2 desc;
```
# result 


using median because of outliers like the 127 items in the cart  would skew the average

| cat| median_purchase_per_c |
|--|--|
|babies|	15 
|international| 15
|missing|	14
|bulk|		14
|canned goods|	14
|dry goods pasta|14
|bakery|	13
|breakfast|	13
|meat seafood| 13
|deli| 13
|snacks|12
|pets| 12
|other|12
|pantry| 12
|frozen| 12
|household| 11
|dairy eggs| 11
|personal care| 11
|produce| 10
|beverages| 10
|alcohol|  7









