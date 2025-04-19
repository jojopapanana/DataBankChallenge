1. How many unique nodes are there on the Data Bank system?
```sql
select count(distinct node_id) as node_count
from customer_nodes
```

Query result:

<img width="146" alt="Screenshot 2025-04-19 at 14 07 36" src="https://github.com/user-attachments/assets/09bde615-8d24-41d2-b4bd-484d8ae5cbcd" />

There are **5 different nodes** in the Data Bank system.


2. What is the number of nodes per region?
```sql
select region_id, count(node_id) as node_count
from customer_nodes
group by region_id
order by region_id;
```

Query result:

<img width="225" alt="Screenshot 2025-04-19 at 14 08 38" src="https://github.com/user-attachments/assets/0ba4efc8-3beb-4f78-9671-8c7f2f36c07c" />

There are 770 nodes in Australia, 735 nodes in America, 714 nodes in Africa, 665 nodes in Asia, and 616 nodes in Europe.


3. How many customers are allocated to each region?
```sql
select region_id, count(distinct customer_id) as customer_count
from customer_nodes
group by region_id
order by region_id;
```

Query result:

<img width="249" alt="Screenshot 2025-04-19 at 14 11 05" src="https://github.com/user-attachments/assets/8a1e329b-57c8-46a7-b058-e524c8bbcc18" />

There are 110 customers in Australia, 105 customers in America, 102 customers in Africa, 95 customers in Asia, and 88 customers in Europe.


4. How many days on average are customers reallocated to a different node?
```sql
select round(avg(end_date - start_date), 2) as average_reallocated_days
from customer_nodes
where end_date != '9999-12-31';
```

Query result:

<img width="221" alt="Screenshot 2025-04-19 at 14 12 14" src="https://github.com/user-attachments/assets/9f3da5e9-ca22-4530-bab9-ec04d1755987" />

The customers are reallocated to a different node in an **average of 14.63 ~ 15 days**.


5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?
```sql
with days_reallocate as(
    select *, age(end_date, start_date) as reallocated_days
    from customer_nodes c join regions r on c.region_id = r.region_id
    where end_date != '9999-12-31'
)

select region_name,
		percentile_cont(0.5) within group(order by reallocated_days) as median,
        percentile_cont(0.8) within group(order by reallocated_days) as percentile_80,
        percentile_cont(0.95) within group(order by reallocated_days) as percentile_95
from days_reallocate
group by region_name;
```

Query result:

<img width="470" alt="Screenshot 2025-04-19 at 14 14 10" src="https://github.com/user-attachments/assets/f6a13aa9-f9d4-441e-a8cb-bdc4d0d46e00" />

The median for all regions is **15 days**, the 80th percentile is either **23 or 24 days**, and the 95th percentile is **28 days** for all region for the average days of reallocation.
