
# 
# FUNCTION 1 - total_payment_amount

```sql
create function total_payment_amount
(
	@customer_id int,   -- Customer ID to filter payments
	@start_date date,   -- Start date for the payment period
	@end_date date      -- End date for the payment period
)
returns float
as
begin
	declare @total_payment float  -- Variable to store total payment amount
	
	set @total_payment = 0;  -- Initialize total payment to zero

	select
		@total_payment = sum(oe.price_per_unit * oe.quantity)  -- Calculate total payment by summing price * quantity
	from order_items as oe
	left join orders as o on o.order_id = oe.order_id
	where o.customer_id = @customer_id and order_date between @start_date and @end_date  -- Filter by customer and date range

	return @total_payment;  -- Return the total payment amount
end;


# USAGE 1

```sql
SELECT dbo.total_payment_amount(50, '2020-01-01', '2024-04-30');



# FUNCTION 2 - fn_customer_order_summary

```sql
create function fn_customer_order_summary
(
	@customer_id int
)
returns @summary table
(
	customer_id int,
	total_spent float,
	order_count int,
	avg_items_per_order float
)
as
begin
	-- Insert the summary data into the table variable
	insert into @summary
	select
		c.customer_id,  -- Customer ID
		isnull(sum(oi.quantity * oi.price_per_unit), 0) as total,  -- Total amount spent by the customer
		count(distinct o.order_id) as order_count,  -- Total number of distinct orders
		case
			when count(distinct o.order_id) = 0 then 0  -- Avoid division by zero
			else cast(sum(oi.quantity) as float)/count(distinct o.order_id)  -- Average number of items per order
		end as avg_items_per_order
	from customers c
	left join orders as o on c.customer_id = o.customer_id
	left join order_items as oi on o.order_id = oi.order_id
	where c.customer_id = @customer_id
	group by c.customer_id;

	return;
end


# USAGE 2

```sql
SELECT * FROM fn_customer_order_summary(76);




# FUNCTION 3 - fn_seller_performance_summary

```sql
create function fn_seller_performance_summary
(
	@seller_int int
)
returns @summary table
(
	seller_id int,
	total_orders int,
	total_sales float,
	total_quantity_sold int,
	avg_product_price float,
	last_order_date date
)
as
begin
	-- Insert performance summary data into the table variable
	insert into @summary
	select
		s.seller_id,  -- Seller ID
		count(distinct o.order_id) as total_orders,  -- Total distinct orders made by the seller
		isnull(sum(oi.quantity * oi.price_per_unit),0) as total,  -- Total sales amount
		isnull(sum(oi.quantity),0) as total_quantity_sold,  -- Total quantity of products sold
		case
			when sum(oi.quantity) = 0 then 0  -- Avoid division by zero
			else cast(sum(oi.price_per_unit * oi.quantity) as float) / sum(oi.quantity)  -- Average product price sold
		end as avg_product_price,
		max(o.order_date) as last_order_date  -- Date of the last order
	from sellers s
	left join orders o on s.seller_id =  o.seller_id
	left join order_items oi on o.order_id = oi.order_id
	where s.seller_id = @seller_int
	group by s.seller_id;

	return;
end;


# USAGE 3

```sql
SELECT * FROM fn_seller_performance_summary(1);


# FUNCTION 4 - fn_customer_behaviour_scorecard

```sql
create function fn_customer_behaviour_scorecard
(
	@customer_id int
)
returns @summary table
(
	customer_id int,
	unique_sellers int,
	avg_order_spend float,
	favorite_category varchar(50),
	delivered_orders int,
	returned_orders int,
	first_order_date date,
	last_order_date date,
	avg_payment_days float
)
as
begin
	-- Insert customer behaviour summary into the table variable
	insert into @summary
	select
		c.customer_id,

		-- Count of distinct sellers the customer bought from
		count(distinct o.seller_id) as unique_sellers,

		-- Average amount spent per order
		case
			when count(distinct o.order_id) = 0 then 0
			else cast(sum(oi.quantity * oi.price_per_unit) as float) / count(distinct o.order_id)
		end as avg_order_spend,

		-- Most frequently ordered product category (favorite_category)
		(
			select top 1 ca.category_name
			from order_items oi2
			left join orders o2 on o2.order_id = oi2.order_id
			left join products p on p.product_id = oi2.product_id
			left join category ca on ca.category_id = p.category_id
			where o2.customer_id = @customer_id
			group by ca.category_name
			order by sum(oi2.quantity) desc
		) as favorite_category,

		-- Number of delivered orders
		count(case when s.delivery_status = 'Delivered' then 1 end) as delivered_orders,
		-- Number of returned orders
		count(case when s.return_date is not null then 1 end) as returned_orders,
		-- Date of first order
		min(o.order_date) as first_order_date,
		-- Date of last order
		max(o.order_date) as last_order_date,
		-- Average days between order date and payment date
		avg(datediff(day, o.order_date, pay.payment_date)*1.0) as avg_payment_days

	from order_items as oi
	left join orders o on o.order_id = oi.order_id
	left join customers c on c.customer_id = o.customer_id
	left join shippings s on s.order_id = o.order_id
	left join payments pay on pay.order_id = o.order_id
	where c.customer_id = @customer_id
	group by c.customer_id;

	return;
end;


# USAGE 4

```sql
select * from fn_customer_behaviour_scorecard(707);


# FUNCTION 5 - fn_customer_order_dynamics

```sql
create function fn_customer_order_dynamics
(
	@customer_id int
)
returns @summary table
(
	order_id int,
	order_date date,
	total_order_amount float,
	total_items int,
	order_rank int,
	spending_category varchar(20)
)
as
begin
	-- CTE to summarize total amount and items per order for the given customer
	;with order_summary as (
		select
			o.order_id,
			o.order_date,
			sum(oi.quantity * oi.price_per_unit) as total_order_amount,  -- Total price for the order
			sum(oi.quantity) as total_items  -- Total items in the order
		from order_items oi
		left join orders o on o.order_id = oi.order_id
		where o.customer_id = @customer_id
		group by o.order_id, o.order_date
		),
	-- CTE to rank orders by total amount and calculate average spend
		ranked_orders as (
			select *,
				rank() over(order by total_order_amount desc) as order_rank,  -- Rank orders by total amount descending
				avg(total_order_amount) over() as avg_spend  -- Average order spend for the customer
			from order_summary
		)
	-- Insert summarized and ranked order data into the return table
	insert into @summary
	select
		order_id,
		order_date,
		total_order_amount,
		total_items,
		order_rank,
		-- Categorize order spending compared to average
		case
			when total_order_amount > avg_spend then 'Above Average'
			when total_order_amount < avg_spend then 'Below Average'
			else 'Equal to AVG'
		end as spending_category
	from ranked_orders;

return;
end;

# USAGE 5

```sql
select * from fn_customer_order_dynamics(707);



# STORED PROCEDURE 1 - AddCustomer

```sql
-- Stored Procedure: AddCustomer
-- Adds a new customer to the customers table with given details.

create procedure AddCustomer
	@CustomerID int,
	@FirstName varchar(20),
	@LastName varchar(20),
	@State varchar(20)
as
begin
	-- Insert new customer record
	insert into customers (customer_id, first_name, last_name, state)
	values(@CustomerID,@FirstName,@LastName,@State);
end;

# USAGE Example - AddCustomer

```sql
exec AddCustomer 2500, 'Ali Veli', 'Cırcıroğlu', 'Aladağ';

# STORED PROCEDURE 2 - AddProduct

```sql
-- Stored Procedure: AddProduct
-- Adds a new product to the products table with specified details.

create procedure AddProduct
	@ProductID int,
	@ProductName varchar(50),
	@Price float,
	@Cogs float,
	@CategoryID int
as
begin
	-- Insert new product record
	insert into products (product_id, product_name, price, cogs, category_id)
	values (@ProductID, @ProductName, @Price, @Cogs, @CategoryID);
end;

# USAGE 2 - AddProduct

```sql
exec AddProduct 5800, 'Nokia 3310', 10, 1, 1;


# STORED PROCEDURE 3 - AddOrder

```sql
-- Stored Procedure: AddOrder
-- Inserts a new order record with details like order date, customer, seller, and status.

create procedure AddOrder
	@OrderID int,
	@OrderDate date,
	@CustomerID int,
	@SellerID int,
	@OrderStatus varchar(15)
as
begin
	-- Insert new order into orders table
	insert into orders (order_id, order_date, customer_id, seller_id, order_status)
	values (@OrderID, @OrderDate, @CustomerID, @SellerID, @OrderStatus);
end;


# USAGE 3 - AddOrder

```sql
exec AddOrder
	25000, '2025-05-12', 707, 5, 'Cancelled';


# STORED PROCEDURE 4 - AddPayment

```sql
-- Stored Procedure: AddPayment
-- Adds a new payment record linked to an order, including payment date and status.

create procedure AddPayment
	@PaymentID int,
	@OrderID int,
	@PaymentDate date,
	@PaymentStatus varchar(150)
as
begin
	-- Insert new payment into payments table
	insert into payments (payment_id, order_id, payment_date, payment_status)
	values (@PaymentID, @OrderID, @PaymentDate, @PaymentStatus);
end;


# USAGE 4 - AddPayment

```sql
exec AddPayment 154000, 1, '2025-07-15', 'Payment Successed';


# STORED PROCEDURE 5 - CancelOrder

```sql
-- Stored Procedure: CancelOrder
-- Cancels an order by updating its status and removing related payment and shipping records.
-- Uses transaction to ensure data integrity.

create procedure CancelOrder
	@OrderID int
as
begin
	begin try
		begin transaction;

		-- Update the order status to 'Cancelled'
		update orders
		set order_status = 'Cancelled'
		where order_id = @OrderID;

		-- Delete payments related to the order
		delete from payments
		where order_id = @OrderID;

		-- Delete shipping info related to the order
		delete from shippings
		where order_id = @OrderID;

		commit transaction;
	end try
	begin catch
		rollback transaction;
		THROW;
	end catch
end;


# USAGE 5 - CancelOrder

```sql
exec CancelOrder 1;


# Stored Procedure: SellProduct

Processes the sale of a product by checking stock, updating inventory,  
and inserting a record into the `order_items` table.  
Uses transaction to maintain consistency and error handling with `THROW`.

```sql
create procedure SellProduct
(
    @OrderID int,
    @ProductID int,
    @Quantity int,
    @PricePerUnit float
)
as
begin
    begin try
        begin transaction;

        -- Check current stock for the product
        declare @CurrentStock int;
        select @CurrentStock = stock from inventory where product_id = @ProductID;
        
        -- If no stock record found, raise an error
        if @CurrentStock is null
        begin
            throw 50001, 'No Found Stock Record', 1;
        end

        -- If stock is insufficient, raise an error
        if @CurrentStock < @Quantity
        begin
            throw 50002, 'Not Enough Stock', 1;
        end

        -- Update inventory to reduce the stock by the sold quantity
        update inventory
        set stock = stock - @Quantity
        where product_id = @ProductID;

        -- Insert new order item record with the next available order_item_id
        insert into order_items (order_item_id, order_id, product_id, quantity, price_per_unit)
        values (
            (select isnull(max(order_item_id), 0) + 1 from order_items),
            @OrderID, @ProductID, @Quantity, @PricePerUnit
        );

        commit transaction;
    end try
    begin catch
        rollback transaction;
        throw;
    end catch
end;

-- USAGE 6

EXEC SellProduct @OrderID = 21629, @ProductID = 3, @Quantity = 2, @PricePerUnit = 249.99;


-- Store Procedure-7  
-- Stored Procedure: sp_customer_purchase_frequency  
-- Calculates the purchase frequency and total amount spent by each customer  
-- within the last 30 days from a fixed date ('2024-07-30').  
-- Orders are counted and total spending is summed.  
-- Results are ordered by purchase count (descending) and total spent (descending).  

```sql
create or alter procedure sp_customer_purchase_frequency
as
begin
	set nocount on;

	declare @days int = 30;

	-- Select customer details with purchase count and total spent in the last 30 days
	select
		c.customer_id,
		c.first_name,
		c.last_name,
		count(o.order_id) as purchase_count,
		sum(oi.quantity * oi.price_per_unit) as total_spent
	from
		order_items oi
	left join orders o on o.order_id = oi.order_id
	left join customers c on c.customer_id = o.customer_id
	where o.order_date >= DATEADD(day, -@days, '2024-07-30') -- fixed max order date for analysis
	group by c.customer_id, c.first_name, c.last_name
	order by purchase_count desc, total_spent desc;

end;


```sql
exec sp_customer_purchase_frequency;


# Stored Procedure: sp_top_valuable_customers

This stored procedure retrieves the top 5 customers based on their total spending within the last 3 months  
from a fixed reference date ('2024-07-30'). It also counts the total number of orders for each customer.

```sql
create or alter procedure sp_top_valuable_customers
as
begin
    set nocount on;

    declare @months int = 3;

    select top 5
        c.customer_id,
        c.first_name,
        c.last_name,
        count(distinct o.order_id) as total_orders,
        sum(oi.quantity * oi.price_per_unit) as total_spent
    from
        order_items as oi
    left join orders o on o.order_id = oi.order_id
    left join customers c on c.customer_id = o.customer_id
    where
        o.order_date >= dateadd(month, -@months, '2024-07-30')  -- fixed reference date; use GETDATE() for dynamic current date
    group by
        c.customer_id, c.first_name, c.last_name
    order by
        total_spent desc;
end;


## Usage Example

```sql
exec sp_top_valuable_customers;

# Trigger: trg_orders_default_status

This trigger automatically sets the `order_status` to `'pending'`  
when a new order is inserted without specifying an `order_status`.  
It runs **after insert** on the `orders` table.

```sql
create or alter trigger trg_orders_default_status
on orders
after insert
as
begin
    set nocount on;

    update o
    set order_status = 'pending'
    from dbo.orders o
    inner join inserted i on o.order_id = i.order_id
    where i.order_status is null or i.order_status = '';
end;

## Example: Select Latest 5 Orders to Verify

```sql
select top 5 * from orders order by order_id desc;


## Example: Insert Order Without Status to Test the Trigger

```sql
insert into orders (order_id, order_date, customer_id, seller_id)
values (21631, '2024-08-31', 707, 26);

## Verify Orders for Customer 707 (Check if Status Is Set)

```sql
select * from orders where customer_id = 707 order by order_date desc;

# Trigger: trg_orders_default_status

This trigger automatically sets the `order_status` to `'pending'` if no status is provided during an insert into the `orders` table.

```sql
create or alter trigger trg_orders_default_status
on orders
after insert
as
begin
    set nocount on;  -- prevent extra result messages

    -- update orders just inserted where order_status is null or empty
    update o
    set order_status = 'pending'
    from dbo.orders o
    inner join inserted i on o.order_id = i.order_id
    where i.order_status is null or i.order_status = '';
end;


## Example: Select latest 5 orders to verify

```sql
select top 5 * from orders order by order_id desc;


## Example: Insert order without status to test the trigger

```sql
insert into orders (order_id, order_date, customer_id, seller_id)
values (21631, '2024-08-31', 707, 26);

## Check orders for customer 707 to confirm status is set

```sql
select * from orders 
where customer_id = 707 
order by order_date desc;













