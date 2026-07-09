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
### Duplicates
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

### Missing values
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

#### Missing values - Item feature  
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

Imputing the missing values in the Item feature by first confirming the relationship between each item and its corresponding Price Per Unit. Once this relationship has been verified, the Price Per Unit and Category of each record with a missing Item value are used to determine the appropriate item for imputation.

```sql
SELECT -- do not update first we will confirm before doing anything 
    r.transaction_id, 
    r.category,
    r.price_per_unit,
    r.item AS old_item,

    l.item AS new_item,

    -- Show the row used as the lookup
    l.category AS lookup_category,
    l.price_per_unit AS lookup_price,
    l.item AS lookup_item

FROM retail_sales_raw r

JOIN (
    SELECT DISTINCT
        category,
        price_per_unit,
        item
    FROM retail_sales_raw
    WHERE item IS NOT NULL
) l
ON r.category = l.category
AND r.price_per_unit = l.price_per_unit

WHERE r.item IS NULL
ORDER BY r.price_per_unit, r.category;
```
<img width="1391" height="407" alt="image" src="https://github.com/user-attachments/assets/ecbbca49-f1dd-4eda-b9c3-0d1c5c5bdf38" />

Then we update and impute the missing values in the Item feature using the verified relationship between Category, Price Per Unit, and Item. After the update, we verify the results once again by counting the remaining missing values in the Item feature.

```sql
UPDATE retail_sales_raw r
SET item = l.item
FROM (
    SELECT DISTINCT
        category,
        price_per_unit,
        item
    FROM retail_sales_raw
    WHERE item IS NOT NULL
) l
WHERE r.item IS NULL
  AND r.category = l.category
  AND r.price_per_unit = l.price_per_unit;
```
<p align = "Center">
  <img width="655" height="147" alt="image" src="https://github.com/user-attachments/assets/ffca0ce4-ee0a-42b0-b136-febdb5e861ab" />
</p>

#### Missing values - Item & Price Per Unit
From 1,213 missing values, we were able to reduce it to only 609. This means that 604 missing values were successfully imputed. The remaining 609 rows could not be imputed for now since they also have missing Price Per Unit values, which we need to determine the corresponding item.

Now let's check for the table again and let's see what we can do from our current information.
```sql
select *
from retail_sales_raw
where item is null
```
<img width="1495" height="401" alt="image" src="https://github.com/user-attachments/assets/e417a1ca-d3e6-480a-88b2-f9410f667fef" />

It can be seen that for the remaining 609 datapoints, both the Item and Price Per Unit features have missing values. However, it can also be seen that these records still have values in the Quantity and Total Spent features. From our earlier observations, we know that Total Spent is equal to Quantity multiplied by Price Per Unit. Therefore, we can rearrange the formula and derive the missing Price Per Unit values by dividing Total Spent by Quantity.

Once the Price Per Unit has been derived, we can then infer the corresponding Item since we have already verified that each price only corresponds to one item number. Lastly, by using the Category feature, we can determine the complete item name.

But first, we need to convert the data types into NUMERIC. Remember that earlier we intentionally set all the columns as TEXT to avoid problems during the import process. Since we are now going to perform arithmetic operations, the Price Per Unit, Quantity, and Total Spent features need to be converted into numeric data types because arithmetic operations cannot be performed on text values.

```sql
ALTER TABLE retail_sales_raw
ALTER COLUMN price_per_unit TYPE NUMERIC
USING price_per_unit::NUMERIC;

ALTER TABLE retail_sales_raw
ALTER COLUMN quantity TYPE NUMERIC
USING quantity::NUMERIC;

ALTER TABLE retail_sales_raw
ALTER COLUMN total_spent TYPE NUMERIC
USING total_spent::NUMERIC;
```

After converting the data types from TEXT to NUMERIC, we can now perform arithmetic operations to calculate the Price Per Unit for each item.
``` sql
UPDATE retail_sales_raw r
SET
    price_per_unit = ROUND(r.total_spent / r.quantity, 1),
    item = l.item
FROM (
    SELECT DISTINCT
        category,
        price_per_unit,
        item
    FROM retail_sales_raw
    WHERE item IS NOT NULL
) l
WHERE r.item IS NULL
  AND r.price_per_unit IS NULL
  AND l.category = r.category
  AND l.price_per_unit = ROUND(r.total_spent / r.quantity, 1);
```

Then, let's verify once again if there are still missing values in the Item feature.

<p align = "center">
  <img width="1042" height="213" alt="image" src="https://github.com/user-attachments/assets/c340f495-59d2-4563-866d-5593417486d1" />
</p>

It can be seen that there are no longer any missing values in the Item feature. By first deriving the missing Price Per Unit values using the Quantity and Total Spent features, we were also able to infer and impute the corresponding Item values in the same update.

#### Missing Values - Quantity and Total Spent
Now regarding the missing values of both Quantity and Total Spent, there are a lot of imputation techniques I am considering:
  - Mean : One of the common imputation techniques when it comes to numerical datasets. However, I do not think this technique is appropriate since it is affected by outliers, and the resulting values would only be estimates rather than the actual values.
  - Mode : Another possible technique is to use the most frequently occurring quantity for each item. However, this assumes that the missing transactions follow the same purchasing behavior as the existing records, which cannot be verified.
  - Median : This is less affected by outliers compared to the mean and is generally a better choice for numerical imputation. However, similar to the previous techniques, the imputed values would still only be estimates and not the actual values.
  - Drop : For now, I think dropping these records would be the best choice since there is really no information to go off on when dealing with the missing values of these features. Unlike the previous missing values, these cannot be derived with certainty. I also think that completeness should not come at the expense of data integrity, especially if it could affect the later analysis.

```sql
DELETE FROM retail_sales_raw
WHERE quantity IS NULL
  AND total_spent IS NULL;
```

And verify:
<p align = "center">
  <img width="803" height="202" alt="image" src="https://github.com/user-attachments/assets/0ec80161-d5bb-495a-9496-9a45edd89d7b" />
</p>

Since we dropped 604 datapoints we are left with only 11,971 datapoints.

Next is converting the Transaction Date feature to a DATE data type as it would be crucial to our next step of preprocessing.
```sql
ALTER TABLE retail_sales_raw
ALTER COLUMN transaction_date
TYPE DATE
USING transaction_date::DATE;
```

#### Missing Values - Discount Applied
Now moving on to Discount Applied, I tried to formulate different hypotheses that might explain how a customer receives a discount.
  - Quantity – Perhaps customers receive a discount when they buy more items.
    - <img width="590" height="401" alt="image" src="https://github.com/user-attachments/assets/c2969512-0c7c-4986-906e-64292e93cec3" />
    - Which seems to not be the case as the discount rate still stays around 50% even as the quantity goes up.
  
  - Payment Method – Another possibility is that the payment method determines whether a discount is applied.
    - <img width="642" height="226" alt="image" src="https://github.com/user-attachments/assets/2dce88e5-4eb1-4938-97c8-d374661319e5" />
    - Which also does not seem to be the case as all payment methods have nearly the same discount rate.
    
  - Location - It could also be that online purchases receive more discounts compared to in-store purchases.
    - <img width="587" height="131" alt="image" src="https://github.com/user-attachments/assets/b93d29bf-104d-4150-a662-f372bf340776" />
    - Which does not seem to be the case as the discount rate is almost a 50/50 split.
    
  - Transaction Date - Another hypothesis is that discounts are offered seasonally, such as during Christmas.
    - <img width="612" height="412" alt="image" src="https://github.com/user-attachments/assets/5fe3755c-8a87-451b-91b4-68920d12b0c4" />
    - Which also does not seem to be the case as the highest discount rate is only 53.66% in September.

  - Total Spent - Lastly, customers who spend above a certain amount may have a higher chance of receiving a discount.
    - <img width="642" height="280" alt="image" src="https://github.com/user-attachments/assets/f37e9b23-6047-4e4e-8148-b4a9a8deafc8" />
    - Which also does not seem to be the case as customers who spent 300+ only have a 52.05% discount rate.

After exploring different hypotheses, none of them showed a strong enough relationship with Discount Applied. The discount rate consistently stayed around 50% regardless of the quantity purchased, payment method, location, transaction date, or total amount spent.

Because of this, there is not enough evidence to confidently infer whether a missing value should be True or False. Rather than filling these missing values with unsupported assumptions, I decided to label them as Unknown. Since this feature contains 3,988 missing values, I believe this is a better approach than dropping those records altogether. It also makes the dataset easier to analyze and visualize later on, as Unknown can be treated as its own category instead of being ignored.

```sql
UPDATE retail_sales_raw
SET discount_applied = 'Unknown'
WHERE discount_applied IS NULL;
```

<img width="450" height="237" alt="image" src="https://github.com/user-attachments/assets/b8c23909-e05e-42f1-afe2-20b9df0680b9" />
