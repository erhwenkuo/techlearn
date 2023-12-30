# Natural language to SQL

![](./assets/default-sql-translate.png)

將自然語言轉換為 SQL 查詢。

Convert natural language into SQL queries.

**Prompt**

|||
|-------|------|
|**SYSTEM**|<pre lang="sql"><code>Given the following SQL tables, your job is to write queries given a user’s request.<br/>    <br/>    CREATE TABLE Orders (<br/>      OrderID int,<br/>      CustomerID int,<br/>      OrderDate datetime,<br/>      OrderTime varchar(8),<br/>      PRIMARY KEY (OrderID)<br/>    );<br/>    <br/>    CREATE TABLE OrderDetails (<br/>      OrderDetailID int,<br/>      OrderID int,<br/>      ProductID int,<br/>      Quantity int,<br/>      PRIMARY KEY (OrderDetailID)<br/>    );<br/>    <br/>    CREATE TABLE Products (<br/>      ProductID int,<br/>      ProductName varchar(50),<br/>      Category varchar(50),<br/>      UnitPrice decimal(10, 2),<br/>      Stock int,<br/>      PRIMARY KEY (ProductID)<br/>    );<br/>    <br/>    CREATE TABLE Customers (<br/>      CustomerID int,<br/>      FirstName varchar(50),<br/>      LastName varchar(50),<br/>      Email varchar(100),<br/>      Phone varchar(20),<br/>      PRIMARY KEY (CustomerID)<br/>    );</code></pre>|
|**USER**|Write a SQL query which computes the average total order value for all orders on 2023-04-01.|

**Sample response**

````
To compute the average total order value for all orders on 2023-04-01, we need to join the Orders, OrderDetails, and Products tables. We will sum the product of Quantity and UnitPrice for each order, and then compute the average of these sums.

Here is the SQL query:

```sql
SELECT AVG(TotalOrderValue) AS AverageOrderValue
FROM (
  SELECT Orders.OrderID, SUM(OrderDetails.Quantity * Products.UnitPrice) AS TotalOrderValue
  FROM Orders
  JOIN OrderDetails ON Orders.OrderID = OrderDetails.OrderID
  JOIN Products ON OrderDetails.ProductID = Products.ProductID
  WHERE Orders.OrderDate = '2023-04-01'
  GROUP BY Orders.OrderID
) AS OrderValues;
```

This query first computes the total value of each order in the subquery, and then computes the average of these values in the outer query.

````

**API request**

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
  "model": "gpt-4",
  "messages": [
    {
      "role": "system",
      "content": "Given the following SQL tables, your job is to write queries given a user’s request.\n    \n    CREATE TABLE Orders (\n      OrderID int,\n      CustomerID int,\n      OrderDate datetime,\n      OrderTime varchar(8),\n      PRIMARY KEY (OrderID)\n    );\n    \n    CREATE TABLE OrderDetails (\n      OrderDetailID int,\n      OrderID int,\n      ProductID int,\n      Quantity int,\n      PRIMARY KEY (OrderDetailID)\n    );\n    \n    CREATE TABLE Products (\n      ProductID int,\n      ProductName varchar(50),\n      Category varchar(50),\n      UnitPrice decimal(10, 2),\n      Stock int,\n      PRIMARY KEY (ProductID)\n    );\n    \n    CREATE TABLE Customers (\n      CustomerID int,\n      FirstName varchar(50),\n      LastName varchar(50),\n      Email varchar(100),\n      Phone varchar(20),\n      PRIMARY KEY (CustomerID)\n    );"
    },
    {
      "role": "user",
      "content": "Write a SQL query which computes the average total order value for all orders on 2023-04-01."
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
