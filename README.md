# mobile store employees and campaign analysis 
 

## Employee Bonuses and Successful Campaigns Analysis


### Introduction 
A small chain of mobile devices is interested in analyzing the employees
sales to identify outstanding employees and successful campaigns.

### The Data
Internal company data of employees, campaigns, and sales


### Step 1:
### Understanding what is in the data with this Query:


	SELECT	schema_name(tab.schema_id) as schema_name
			,tab.name as table_name
			,col.column_id
			,col.name as column_name
			,t.name as data_type    
			,col.max_length
			,col.precision
	FROM sys.tables as tab
	INNER JOIN sys.columns as col
	ON tab.object_id = col.object_id
	LEFT JOIN sys.types as t
	ON col.user_type_id = t.user_type_id
	ORDER BY schema_name
		,table_name 
		,column_id;

### 1. EMPLOYEE BONUSES


- Employee id
- Age
- (Number of orders excluding order totals of over 1,000 NIS (and excluding canceled orders)0
- Indication if the employee is active
- Number of unique costumers for each employee
- Number of days that passed since the last order of each employee
- Total average order for each employee after discounts and cancelations

### Tables for Solution:
1.	Employees – in the table 20 employee id's which represent each employee and his/her personal information
2.	Orders – in the table each order id is connected to an employee, customer, and a lead 
		(some employees, customers and leads have multiple orders)
3.	Orders_detailed – full details for each order, quantity, items, discounts and status
4.	Items – name and price for each item ID


		WITH CTE AS
		(
			SELECT	 e.Employee_ID 
				,COUNT(DISTINCT o.Customer_ID) AS 'unique_customers_per_emp'
				,COUNT(DISTINCT o.Order_ID) AS 'unique_orders_per_emp'
			FROM employees e
			LEFT JOIN orders o
			ON e.Employee_ID = o.Employee_ID
			GROUP BY e.Employee_ID
		),CTE_2 AS
		(
			SELECT	 e.*
			,o.Customer_ID
			,o.Lead_ID
			,o.Date
			,od.*
			,DATEDIFF(year,e.Employee_Birthday,getdate()) AS age
			,CASE WHEN e.date_of_work_end < getdate() THEN 0    
				  WHEN e.date_of_work_end IS NULL THEN 1
				  ELSE 1
			 END AS active_emp	
			,datediff(d, o.date, getdate()) AS days_last_order
			,c.unique_customers_per_emp
			,c.unique_orders_per_emp
			,CASE WHEN od.Discount > 0 AND od.Status IS NULL THEN F7 *(1 - od.discount)
			      WHEN od.Discount = 0 AND od.Status IS NULL THEN F7
			      ELSE ''
			 END AS 'price'
			,SUM(CASE WHEN od.Discount > 0 AND od.Status IS NULL THEN F7 *(1 - od.discount)
				  WHEN od.Discount = 0 AND od.Status IS NULL THEN F7
				  ELSE NULL
			 END) OVER (PARTITION BY(o.Order_id)) AS 'sum_per_order'
			,AVG(CASE WHEN od.Discount > 0 AND od.Status IS NULL THEN F7 *(1 - od.discount)
				  WHEN od.Discount = 0 AND od.Status IS NULL THEN F7
				  ELSE NULL
			 END) OVER (PARTITION BY(o.Order_id)) AS 'avg_per_order'
			,CASE WHEN SUM(CASE WHEN od.Discount > 0 AND od.Status IS NULL THEN F7 *(1 - od.discount)
			      WHEN od.Discount = 0 AND od.Status IS NULL THEN F7
			      ELSE ''
			      END) OVER (PARTITION BY(o.Order_id)) >= 1000 THEN 1 
			      ELSE 0
			 END AS 'indication'
			FROM CTE c
			JOIN employees e
			ON c.Employee_ID = e.Employee_ID
			LEFT JOIN orders o
			ON e.Employee_ID = o.Employee_ID
			LEFT JOIN Orders_Detailed od
			ON o.Order_ID = od.Order_ID
		)
			SELECT	 Employee_ID
				,MAX(age) AS 'emp_age'
				,MAX(active_emp) AS 'emp_active'
				,MAX(unique_customers_per_emp) AS 'total_customers_per_emp'
				,MAX(days_last_order) AS 'days'
				,SUM(price) AS 'total_sales_by_emp'
				,CASE WHEN MAX(unique_orders_per_emp) > 0 THEN SUM(price) / MAX(unique_orders_per_emp)
				      ELSE 0
				 END AS 'avg_sales_by_emp'
			FROM CTE_2
			GROUP BY Employee_ID
			ORDER BY SUM(price) DESC

### SUCCESSFUL CAMPAINS


- Campaign_ID
- Unique leads per campaign
- Unique employees per campaign
- Indication for an iPhone bought in each campaign
- Average Total per order for each campaign after discounts and cancelations
- Total of all orders for each campaign after discounts and cancelations
- Number of days sinch last order in each campaign
- Order campaign by days sinch last order descending

### Tables for Solution
1.	Campaigns – in the table 20 campaign ID's  but, one campaign id is a duplicate , a total of 19 unique campaign ID's. 
	the Duplicate Campaign ID can lead to miss calculation when joining tables
2.	Leads – in the table 20 lead ID's that are connected to 15 unique campaigns 
3.	Orders – in the table each order id is connected to an employee, customer, 
	and a lead (some employees, customers and leads have multiple orders)
4.	Orders_detailed – full details for each order, quantity, items, discounts, and status
5.	Items – name and price for each item ID



		WITH CTE AS
		(
			SELECT   C.Campaign_ID
					,l.lead_id
					,COUNT(DISTINCT L.Lead_ID) 'leads_per_camp'
			FROM Campaigns c
			LEFT JOIN leads l
			ON c.Campaign_ID = l.Campaign_ID
			GROUP BY C.Campaign_ID, l.lead_id
		),CTE_2 AS
		(
		SELECT	 c.*
				,o.Order_ID
				,o.Employee_ID
				,o.Customer_ID
				,o.Date
				,od.Item_ID
				,od.Quantity
				,od.Discount
				,od.Status
				,i.Item_Name
				,i.Item_Price
				,SUM(c.leads_per_camp) OVER(PARTITION BY Campaign_ID) AS 'sum_leads_per_camp'
				,COUNT(Employee_ID) OVER(PARTITION BY Campaign_ID) AS 'emps_per_lead'
				,CASE WHEN i.Item_Name LIKE '%iphone%' THEN 1
						  ELSE 0
				 END AS 'iphone_in_order'
				,CASE WHEN Discount  > 0 THEN F7 * (1.00 - CAST(LEFT(od.Discount, 4) AS float)/100)
					  ELSE  F7 
				 END AS 'price'
		,DATEDIFF(d, o.Date, GETDATE()) AS days_last_order
		FROM CTE c
		LEFT JOIN orders o
		ON c.Lead_ID = o.Lead_ID
		LEFT JOIN Orders_Detailed od
		ON  o.Order_ID = od.Order_ID
		LEFT JOIN items i
		ON od.Item_ID = i.Item_ID
		)
		SELECT	 Campaign_ID
				,MAX(sum_leads_per_camp) 'leads_camp'
				,MAX(emps_per_lead) 'emps_camp'
				,MAX(iphone_in_order) 'iphone_in_camp'
				,AVG(price) 'avg_per_camp'
				,SUM(price) 'total_per_camp'
				,MIN(days_last_order) 'last_order_in_camp'		
		FROM CTE_2
		GROUP BY Campaign_ID
		ORDER BY MIN(days_last_order) DESC

## INSIGHTS and THOUGHTS


### 1. Employees
a. age doesnt have an impact
b. who are the employees that leave the company? 
   The not affective ones, no/low total/average sales 
   or the good ones  the opposite, many customers/high total, and average sales) 
   get taken a way by other job opportunities
c. ability to identify employees deserving bonuse, Would recomand by top average per order
d. if it's a commission based salery then calculate employee's percentage per sale

### 2.	Campaigns
a. rate best campaign according to highest total of sales
b. rate best campaign according to number of leads per sale 
   (the less leads that end in a sale = a better campaign)
c. rate best campaign according to the number of employees involved in each sale
   (campaign 452 has a total of 13,885 (average 2314 )but also 6 different employees linked to it, 
   campaigns 566, 517 and 355 have higher averages and only one employee linked to the campaign)

