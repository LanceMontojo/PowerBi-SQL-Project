# ETL with PowerBi

Because this dataset is intentionally messy, I wanted to document my exact thought process and steps for cleaning it. This is a breakdown of my systematic approach to handling the "dirty" work—from managing missing values and removing duplicates to getting the data dashboard-ready.

## Extract
For this project, the dataset is stored in a PostgreSQL database instead of being imported directly from a CSV file. Although the original dataset is in comma-separated values (CSV) format, connecting Power BI to PostgreSQL simulates a more realistic workflow where data is retrieved from a relational database.

To connect the dataset to Power BI:
- Click Home → Get Data → More.
- Select PostgreSQL Database.
- Enter the Server (localhost) and the appropriate Database name from PostgreSQL.
- Choose Import as the connectivity mode and click OK.
- Select the raw retail sales table from the Navigator, then click Transform Data to open the Power Query Editor, where the ETL process will be performed.

Instead of importing a CSV file directly, the dataset was stored in a PostgreSQL database and connected to Power BI using the PostgreSQL connector. 

## Transform

Since we already identified the appropriate data types during the SQL ETL process, we can apply the same changes in Power Query. Verifying and assigning the correct data types is one of the first transformations performed because subsequent operations such as filtering, sorting, calculations, and aggregations rely on columns being interpreted correctly.

### Data Types
To change a column's data type:
- Click the data type icon beside the column name and select the appropriate data type (e.g., Text, Whole Number, Decimal Number, Date, or Boolean).
- Alternatively, select the column and navigate to the Transform tab, then choose Data Type.
- Change data types of:
  - Price Per Unit to be Decimal Number
  - Quantity to be Whole Number
  - Total Spent to be Decimal Number

<img width="1167" height="37" alt="image" src="https://github.com/user-attachments/assets/5d50c855-594c-4331-94c6-6198c7a0f00e" />

After changing the data types, notice that Power Query automatically creates a Changed Type step under Applied Steps. Unlike SQL, where transformations are written as a script, Power Query records each transformation as an individual step. This allows any ETL step to be modified, reordered, or removed without rewriting the entire workflow.

<p align = "center">
  <img width="307" height="172" alt="image" src="https://github.com/user-attachments/assets/0e818600-cc8a-44bf-8c22-f1bd92b40ec0" />
</p>

For consistency while inspecting the data, the dataset was sorted by Transaction Date in descending order by selecting the drop-down arrow on the Transaction Date column and choosing "Sort Descending".

Now let's check for missing values. One of the advantages of Power Query is that it has built-in data profiling tools that make it much easier to inspect the quality of a dataset. Instead of writing multiple SQL queries to count missing values, identify duplicates, or check unique values, Power Query summarizes this information visually for every column.

Profiling the dataset before cleaning helps us understand its current condition and determine which transformations are actually necessary instead of making unnecessary changes.

To enable data profiling:
- Go to the "View" Tab
- Enable:
  - Column Quality
  - Column Distribution
  - Column Profile

Column quality displays the percentage of Valid, Error, and Empty(null) values giving us a quick summary of which columns contain missing or invalid values, making it easier to identify where cleaning is required. Column Distribution displays the Distinct (A value that appears at least once in a column) and Unique (A value that appears exactly once in a column) values that is useful for identifying inconsistent categories, unexpected values, and possible duplicate records.

<img width="1371" height="202" alt="image" src="https://github.com/user-attachments/assets/0c2e5f17-f0fc-458d-8c4e-5d518f7945da" />

As can be seen the summary of missing values, distinct values and more are visible once data profiling is enabled. Features such as Item, Price Per Unit, Quantity, Total Spent, and Discount Applied all contains null values. 

### Missing Values

Since we already know that:
- Total Spent can be calculated by multiplying Price Per Unit and Quantity. However, because both Price Per Unit and Quantity also contain missing values, this calculation is only possible when the other required value is available.
- Quantity can be calculated by dividing Total Spent by Price Per Unit, provided that both values are present.
- Price Per Unit can be inferred from the Item column since every item has a consistent selling price throughout the dataset. For example, if Item_10_PAT always has a price of 18.5, then any missing Price Per Unit for that item can be replaced with 18.5.
- Likewise, Item can be inferred using a combination of Category and Price Per Unit. For example, a record belonging to the Patisserie category with a Price Per Unit of 18.5 can be identified as Item_10_PAT.

We will deal with the relationship between Item and Price Per Unit first where a corresponding Item has a corrresponding Price Per Unit. 
Lets verify once again using Power Query and duplicate the current table as to avoid changing our original dataset
- Duplicate the table.
- Rename it (for documentation purposes and to make it easier to identify things)
- Keep only Item and Price Per Unit features and remove all other columns as these two are the ones we will confirm the relationship with.
- Filters with "Begins With" and click "Advanced" and opt for OR.

As can be seen each distinct item only has a corresponding distinct value. 

<img width="467" height="728" alt="image" src="https://github.com/user-attachments/assets/13f72c68-d100-422e-8dd2-eaa4eb4a72e9"/>

#### Missing Values - Items

Now we will deal with missing values in the Item feature.
- On the Queries pane, duplicate your main table and rename it to something like Item Cleaning.
<img width="223" height="117" alt="image" src="https://github.com/user-attachments/assets/64d7e79a-0f3c-41d9-898b-a22b3cfe740d" />

- Keep only the Category, Item, and Price Per Unit features, then remove all other features.
<img width="707" height="703" alt="image" src="https://github.com/user-attachments/assets/7822c01b-b25d-4049-8435-ad32e1232482" />

- Filter out the null values in the Item feature, then remove duplicate rows.
<img width="306" height="246" alt="image" src="https://github.com/user-attachments/assets/7d42035f-914f-400f-b92e-8bee5e765083" />

- Go back to the main table, navigate to the Home tab, then click Merge Queries.

- Hold the Ctrl key and select both the Category and Price Per Unit features in each table. Keep the Join Kind as Left Outer, then click OK.
<img width="885" height="796" alt="image" src="https://github.com/user-attachments/assets/01133e47-0e7c-4f5a-b2f3-88399e8f093d" />


- Expand the merged column, select only the Item feature, and uncheck Use original column name as prefix.

- Reorder the columns so that both Item features are beside each other.

- Go to the Add Column tab, click Conditional Column, and give it a name.

  - Configure as:
  <img width="1163" height="492" alt="image" src="https://github.com/user-attachments/assets/fe3969c4-8d20-4e5b-b96d-bd64c31e7de5" />
  
- Reorder the columns once more.

From 1,213 missing values, only 609 remain. Finally, remove the original Item columns, keep only the newly created Item feature, and rename it back to Item.

<img width="310" height="260" alt="image" src="https://github.com/user-attachments/assets/6fdc0d3f-08aa-438f-a24f-28dbb806a5fa" />

<img width="317" height="248" alt="image" src="https://github.com/user-attachments/assets/0a9a961b-d5ea-47e0-99e5-6b46ac3233b6" />

#### Missing values - Price Per Unit
Since we derived the Item feature from Price Per Unit, the remaining missing values in Item also have missing values in Price Per Unit. Therefore, we need to determine the missing Price Per Unit values first.

We know that Total Spent is calculated by multiplying Price Per Unit by Quantity. Rearranging the formula allows us to infer the missing Price Per Unit values by dividing Total Spent by Quantity.

To do this:
- Go the "Add Column" tab and click "Custom Column".

- Name the column and then put the formula of:

  - <img width="875" height="527" alt="image" src="https://github.com/user-attachments/assets/f3f07c6e-e6ce-44d9-a2cf-a47c810a4bf3" />

- Remove the original Price Per Unit feature and rename the newly created feature back to Price Per Unit.

<img width="307" height="226" alt="image" src="https://github.com/user-attachments/assets/c3a87ce0-b0a6-494f-b44b-d3d3e617111f" />

Now that the Price Per Unit feature no longer contains missing values, the Item feature can be imputed once again using the same merge process described previously. Since the lookup depends on both Category and Price Per Unit, the newly recovered price values allow additional missing Item values to be inferred. This reduces the remaining missing values in the Item feature even further.

Redo the Item section of missing values. Now both Price Per Unit and Item no longer contains missing values.

<img width="367" height="285" alt="image" src="https://github.com/user-attachments/assets/83b9c7e4-3386-4a81-a070-ddf1c460c513" />

<img width="330" height="273" alt="image" src="https://github.com/user-attachments/assets/8c7138a9-1b06-4345-9e49-61c4a3e1591e" />


