1. What is the unique count and total amount for each transaction type?
```sql
select txn_type, count(*) as transaction_count, sum(txn_amount) as total_amount
from customer_transactions
group by txn_type;
```

Query results:

<img width="433" alt="Screenshot 2025-04-19 at 14 19 42" src="https://github.com/user-attachments/assets/6549a196-6d5a-48b4-8d4b-ce5ebabc0a5c" />

From the query results, it can be seen that most of the transactions happened are of **Deposit** type and that also caused the amount of **Deposit** to be the **highest**.

2. What is the average total historical deposit counts and amounts for all customers?
```sql
select txn_type, round(avg(txn_count), 2) as avg_count, round(avg(txn_sum), 2) as avg_amount
from(
	select customer_id, txn_type, count(*) as txn_count, sum(txn_amount) as txn_sum
  	from customer_transactions
  	where txn_type like 'deposit'
  	group by customer_id, txn_type
) c
group by txn_type;
```

Query results:

<img width="384" alt="Screenshot 2025-04-19 at 14 22 44" src="https://github.com/user-attachments/assets/ac07e2e2-154e-4992-af6f-492ca8e4a88a" />

Among all customers, the average historical deposit count is **5.34**, which means for each customer who has done minimum 1 deposit, they have done deposits on average 5 times, and the average amount is **2718.34**, which means for each customer who has done minimum 1 deposit, the average of their deposit amount is 2718.

3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
```sql
with transaction_types as(
	select extract(month from txn_date) as transaction_month, customer_id,
  			count(case when txn_type like 'deposit' then 1 else 0 end) as deposit_count,
            count(case when txn_type like 'purchase' then 1 else 0 end) as purchase_count,
            count(case when txn_type like 'withdrawal' then 1 else 0 end) as withdrawal_count
  	from customer_transactions
  	group by transaction_month, customer_id
)

select transaction_month, count(customer_id) as customer_count
from transaction_types
where deposit_count > 1 and (purchase_count > 0 or withdrawal_count > 0)
group by transaction_month
order by transaction_month;
```

Query results:

<img width="303" alt="Screenshot 2025-04-19 at 14 27 06" src="https://github.com/user-attachments/assets/dd85d125-761e-4ae9-bf2f-826dc38752cb" />

The amount of customers who did the stated transaction conditions increased from the 1st to the 3rd month, but decreased on the 4th month.

4. What is the closing balance for each customer at the end of the month?
```sql
with transaction_amount as(
	select extract(month from txn_date) as transaction_month, customer_id, sum(case when txn_type like 'deposit' then txn_amount else -txn_amount end) as amount
  	from customer_transactions
  	group by transaction_month, customer_id
)

select transaction_month, customer_id, sum(amount) over(partition by customer_id order by transaction_month rows between unbounded preceding and current row) as closing_balance
from transaction_amount
group by transaction_month, customer_id, amount
order by transaction_month, customer_id;
```

Query results:

<img width="400" alt="Screenshot 2025-04-19 at 14 30 11" src="https://github.com/user-attachments/assets/57bae0a1-f3f3-4e89-853e-07690d70db7d" />

What is the percentage of customers who increase their closing balance by more than 5%?
```sql
with monthly_amount as(
	select extract(month from txn_date) as transaction_month, customer_id, sum(case when txn_type like 'deposit' then txn_amount else -txn_amount end) as amount
  	from customer_transactions
  	group by transaction_month, customer_id
),
monthly_closing_balance AS(
   SELECT 
   	customer_id, 
   	transaction_month,
	amount,
   	sum(amount) over(partition by customer_id order by transaction_month rows between unbounded preceding and current row) as closing_balance
   FROM monthly_amount
   GROUP BY customer_id, transaction_month, amount
   ORDER BY customer_id
),
balance_percentage_increase AS (
   SELECT 
   	customer_id,
   	transaction_month,
   	closing_balance,
   	100 *(closing_balance - LAG(closing_balance) OVER(PARTITION BY customer_id ORDER BY transaction_month))
   		 / NULLIF(LAG(closing_balance) OVER(PARTITION BY customer_id ORDER BY transaction_month), 0) AS percentage_increase
   FROM monthly_closing_balance
)

select count(distinct customer_id) * 100 / (select count(distinct customer_id) from customer_transactions) as more_than_5percent_increase
from balance_percentage_increase p
where p.percentage_increase > 5
```

Query results:

<img width="243" alt="Screenshot 2025-04-19 at 14 31 41" src="https://github.com/user-attachments/assets/fa75860c-46ea-4cf2-ae16-bbdc5284593f" />

There are **75% of the customers** who have more than 5% increase in their balance since the first they joined. 
