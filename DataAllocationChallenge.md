To test out a few different hypotheses - the Data Bank team wants to run an experiment where different groups of customers would be allocated data using 3 different options:

Option 1: data is allocated based off the amount of money at the end of the previous month

Option 2: data is allocated on the average amount of money kept in the account in the previous 30 days

Option 3: data is updated real-time

For this multi-part challenge question - you have been requested to generate the following data elements to help the Data Bank team estimate how much data will need to be provisioned for each option:

1. Running customer balance column that includes the impact each transaction
```sql
with monthly_amount as(
  select txn_date, customer_id, case when txn_type like 'deposit' then txn_amount else -txn_amount end as amount
  from customer_transactions
)

select customer_id, txn_date, sum(amount) over(partition by customer_id order by txn_date rows between unbounded preceding and current row) as running_balance
from monthly_amount
group by txn_date, customer_id, amount
order by customer_id
```

Query results:

<img width="346" alt="Screenshot 2025-04-19 at 14 36 59" src="https://github.com/user-attachments/assets/1941d4f5-cb8c-4c9c-b58c-16c1d0e3fb46" />

2. Customer balance at the end of each month
```sql
with transaction_amount as(
	select extract(month from txn_date) as transaction_month, customer_id, sum(case when txn_type like 'deposit' then txn_amount else -txn_amount end) as amount
  	from customer_transactions
  	group by transaction_month, customer_id
)

select transaction_month, customer_id, sum(amount) over(partition by customer_id order by transaction_month rows between unbounded preceding and current row) as closing_balance
from transaction_amount
order by customer_id, transaction_month
```

Query results:

<img width="401" alt="Screenshot 2025-04-19 at 14 38 27" src="https://github.com/user-attachments/assets/5cdb3186-cda2-4a62-9f5c-2af979f21b4c" />

3. Minimum, average and maximum values of the running balance for each customer
```sql
with transaction_amount as(
	select txn_date, customer_id, case when txn_type like 'deposit' then txn_amount else -txn_amount end as amount
  	from customer_transactions
), cust_running_balance as(
	select customer_id, sum(amount) over(partition by customer_id order by txn_date rows between unbounded preceding and current row) as running_balance
	from transaction_amount
  	group by customer_id, amount, txn_date
)

select customer_id, min(running_balance) as min_running_balance, round(avg(running_balance), 2) as avg_running_balance, max(running_balance) as max_running_balance
from cust_running_balance
group by customer_id
```

Query results:

<img width="588" alt="Screenshot 2025-04-19 at 14 39 18" src="https://github.com/user-attachments/assets/cfa9722e-2709-40a2-a321-31e86e242b6c" />

Using all of the data available - how much data would have been required for each option on a monthly basis?
- Option 1:
```sql
with transaction_amount as(
	select extract(month from txn_date) as transaction_month, customer_id, sum(case when txn_type like 'deposit' then txn_amount else -txn_amount end) as amount
  	from customer_transactions
  	group by transaction_month, customer_id
),
running_monthly_balance as(
    select transaction_month, customer_id, sum(amount) over(partition by customer_id order by transaction_month rows between unbounded preceding and current row) as closing_monthly_balance
    from transaction_amount
    order by customer_id, transaction_month
),
prev_month as(
	select *, lag(closing_monthly_balance, 1) over(partition by customer_id order by customer_id, transaction_month) as prev_month_balance
  	from running_monthly_balance
  	order by transaction_month
)

select transaction_month, sum(case when prev_month_balance < 0 then 0
                              when prev_month_balance is null then closing_monthly_balance 
                              else prev_month_balance end) as total_allocation
from prev_month
group by transaction_month
order by transaction_month
```

Query results:

<img width="298" alt="Screenshot 2025-04-19 at 14 40 25" src="https://github.com/user-attachments/assets/c7f602f8-77f9-4320-9815-8daa49fa7724" />

Above is the data allocation needed each month based on the amount of money at the end of the previous month.

- Option 2:
```sql
with transaction_amount as(
	select extract(month from txn_date) as transaction_month, customer_id, sum(case when txn_type like 'deposit' then txn_amount else -txn_amount end) as amount
  	from customer_transactions
  	group by transaction_month, customer_id
),
running_monthly_balance as(
    select transaction_month, customer_id, sum(amount) over(partition by customer_id order by transaction_month rows between unbounded preceding and current row) as closing_monthly_balance
    from transaction_amount
    order by customer_id, transaction_month
),
avg_balance as(
	select transaction_month, customer_id, avg(closing_monthly_balance) over(partition by customer_id) as avg_running_balance
  	from running_monthly_balance
  	group by transaction_month, closing_monthly_balance, customer_id
	order by transaction_month, customer_id
)

select transaction_month, round(sum(case when avg_running_balance < 0 then 0 else avg_running_balance end), 2) as data_allocated
from avg_balance
group by transaction_month
order by transaction_month
```

Query results:

<img width="295" alt="Screenshot 2025-04-19 at 14 44 19" src="https://github.com/user-attachments/assets/67f9d362-da23-47fe-8d72-6e21dfba83f6" />

Above is the data allocation needed each month based on the average amount of money in the previous 30 days.

- Option 3:
```sql
with transaction_amount as(
	select extract(month from txn_date) as transaction_month, customer_id, sum(case when txn_type like 'deposit' then txn_amount else -txn_amount end) as amount
  	from customer_transactions
  	group by transaction_month, customer_id
),
running_monthly_balance as(
    select transaction_month, customer_id, sum(amount) over(partition by customer_id order by transaction_month rows between unbounded preceding and current row) as closing_monthly_balance
    from transaction_amount
    order by customer_id, transaction_month
)

select transaction_month, sum(case when closing_monthly_balance < 0 then 0 else closing_monthly_balance end) as data_allocated
from running_monthly_balance
group by transaction_month
order by transaction_month
```

Query results:

<img width="295" alt="Screenshot 2025-04-19 at 14 45 37" src="https://github.com/user-attachments/assets/88e63539-278f-4264-83f1-53a825752598" />

Above is the data allocation needed each month based on the amount of money updated real-time.
