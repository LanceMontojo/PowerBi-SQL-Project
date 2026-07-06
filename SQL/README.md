# ETL with PostgreSQL

Because this dataset is intentionally messy, I wanted to document my exact thought process and the steps I took to clean it. 
This section outlines my systematic approach to handling the "dirty" work, including managing missing values and removing duplicates and more
using PostgreSQL.

## Extract
The dataset is provided in a Comma-Separated Values (CSV) file, making it straightforward to import into PostgreSQL. 
The extraction process consisted of the following steps:
- Create a database named retail_sales_raw, as the dataset is still in its raw form and has not yet undergone preprocessing. 

- Create a table within the database, either by manually defining the columns or by using SQL commands such as **CREATE TABLE**.
Initially assign all columns with **TEXT** data type to all columns. Doing it this way ensures that PostgreSQL 
imports the data exactly as it appears in the CSV file and prevents import failures caused by datatype mismatches, invalid values, or missing data.

- Verify that the import was successful by using **COUNT** function to check whether the number of rows in the table matches the number of 
records in the original dataset.

<p align = "center">
  <img width="167" height="82" alt="image" src="https://github.com/user-attachments/assets/c07a977e-96c3-4814-819a-eb0b68f1ed1c" />
</p>

Since the count matches our dataset, we can safely assume the import was successful. The first five datapoint of our dataset looks like:

<img width="1452" height="203" alt="image" src="https://github.com/user-attachments/assets/a9214f1a-1d0e-4d32-a9ea-c57d7191ee46" />

## Transform
- After obtaining the dataset, the first step should be assessing the data quality by checking for duplicates. By using the **COUNT** function, duplicate valeus can be identified and investigated. In this specific dataset, Transaction_ID seems to be the unique identifier, so it is useful to know if there are duplicated values of data points in that column. The reasoning I have for this is that

  - Customer_ID may contain duplicate values because a customer can make multiple purchases over time.
  - Transaction_Date may contain duplicate values because many transactions can occur on the same day
  - Category contains a fixed set of product categories that are shared across many transactions.
  - Item cannot be a unique identifier either because the same item can be purchased by different customers and across multiple transactions. For example,       Item_10_PAT belongs to the Patisserie category and can appear in many records.
  - Price_Per_Unit is not unique because multiple transactions may involve the same item at the same price.
  - Quantity is simply the number of units purchased and can naturally repeat across transactions.
  - Payment_Method, Location, and Discount_Applied are descriptive attributes and are expected to have many repeated values.

 - Based on this reasoning, Transaction_ID is the most appropriate column to check for duplicate records. If duplicate Transaction_IDs are found, they may indicate duplicate transaction entries in the dataset.

<p align = "Center">
  <img width="738" height="157" alt="image" src="https://github.com/user-attachments/assets/9294b178-2cd3-49af-b18f-89a7f3ad60bd" />
</p>

This verifies that there are no duplicate values and therefore we can move to the next step.

- For the next step of data assessment, we check for missing values across all features in the dataset.
<img width="1457" height="95" alt="image" src="https://github.com/user-attachments/assets/bdf1b4a7-9d2a-44cc-b950-ad210165e282" />

<p align = "center">
  <img width="570" height="87" alt="image" src="https://github.com/user-attachments/assets/39f6bf24-0b96-4130-8d59-7e5ea64de14a" />
</p>

<p align = "center">
  <img width="670" height="281" alt="image" src="https://github.com/user-attachments/assets/51b0c6ab-10c5-407e-b809-217d95dcbc62" />
</p>

As shown in the results, the features Item, Price_Per_Unit, Quantity, Total_Spent, and Discount_Applied contain missing values. Since these fields are important for understanding transaction details and sales performance, further investigation is required before deciding how to handle the missing data.

- The next step is to inspect these missing values and determine the most appropriate treatment method, From the table it can be seen that:
  - Total Spent can be calculated by multiplying Price Per Unit and Quantity. However, both the Price Per Unit and Quantity columns contain missing values, making direct imputation difficult in some cases.
  - Quantity can be derived by dividing Total Spent by Price Per Unit whenever both values are available.
  - Price Per Unit can be inferred from the Item column. Ex. Category is Patisserie and Item_10_PAT price per unit is 18.5 so all item_10_pat have price of 18.5
  - Item values may also be inferred using a combination of Category and Price Per Unit. For example, a record in the Patisserie category with a price per unit of 18.5 can be assigned the item value Item_10_PAT.
  - For now, I would skip the Discount_Applied feature and come back to it later. My hope is that some of the inconsistencies observed so far can be explained once the relationships between Item, Price Per Unit, Quantity, and Total Spent have been fully established.
  
From the observations thus far, the easiest issue to address is the relationship between Item and Price_Per_Unit. The hypothesis is that each Item corresponds to a single price. To verify this, a query was executed to identify any items associated with more than one distinct price.

 <p align = "center">
   <img width="845" height="238" alt="image" src="https://github.com/user-attachments/assets/f4cfc49f-ca77-45d8-8dc0-4814e3962b36" />
 </p>

With it returning zero row, it confirms that no item is associated with more than one price.

To further validate this relationship across the dataset, the number of distinct prices associated with each item was examined.
<img width="940" height="412" alt="image" src="https://github.com/user-attachments/assets/0776bf8c-5780-4a0d-ae86-7df58e77ef41" />

The results show that every item is associated with exactly one distinct price. Therefore, it is reasonable to use the relationship between item and price_per_unit when imputing missing values. For example, Item_1 always corresponds to a price of 5.0, Item_10 always corresponds to 18.5, and so on and so forth.

As a result, records with missing item values can be inferred using their corresponding category and price information. For instance, a record belonging to the Patisserie category with a price corresponding to 18.5 can be assigned the Item value Item_10_PAT.
<img width="1505" height="402" alt="image" src="https://github.com/user-attachments/assets/3ca616bb-7d2c-4fd1-b8cd-ba6a09b442df" />

For now I would rest and the plan for the next session would be imputing values from the Item section and hopefully covers more answer so thar we can clean the dataset.
