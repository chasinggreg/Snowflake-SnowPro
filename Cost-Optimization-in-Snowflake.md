# Optimizing Snowflake without compromising performance

Snowflake charges for Data Storage and Compute for any operation in the Snowflake.

## Selection for the right Snowflake instance.

Snowflake enables businesses to manage vast amounts of data efficiently. However, it's important to manage costs effectively, as Snowflake's pricing is usage-based. This article outlines practical strategies to optimize Snowflake usage and reduce costs without sacrificing performance.

Snowflake offers three types of instances: Standard, Enterprise, and Business Critical, available on three significant cloud providers: AWS, Azure, and GCC. Selecting the appropriate Snowflake instance is crucial, as this choice can yield substantial cost savings.

The cost varies per credit across the Standard, Enterprise, and Business Critical plans. To provide a reference, a single credit on the Standard plan is priced at $2.6, whereas the identical credit on the Enterprise plan amounts to approximately $3.7. It’s important to note that the Enterprise plan offers supplementary functionalities such as row-level security and a 30-day time travel feature. Depending on your specific requirements, I recommend conducting thorough research to identify the most suitable instance for your needs.

Choosing a cloud provider and region is of utmost importance, given that cloud providers typically impose charges for cross-region and cross-cloud provider data transfers. Conduct thorough research to determine the cloud provider and region hosting other services within your company. Based on that choose your Snowflake Instance.

In the context of our company, all applications are hosted on AWS. After assessing our requirements, we determined that the standard version meets our needs. As a result, we opted to proceed with the standard Snowflake instance hosted on AWS.

Cost saving on the storage side.

In this section, I will describe the changes we implemented as part of Data Organization and Storage.

<ol>
<li><b>Creation of different environments with cloning.</b>
Instead of populating the data from the data pipeline into different environments such as dev and stg, utilize Snowflake’s cloning feature. We clone our dev and stg environments from prod over the weekend. This approach allows us to save costs on storage, as we only incur charges for storing the modified or altered records.
It is just a single-line command to clone the database.</li>

`CREATE OR REPLACE DATABASE my_STG CLONE my_PROD;`

Here, we are establishing a database named my_STG using the my_PROD as a source. Until you make any modifications to the my_STG database, you won’t incur additional storage charges. One issue that we found in this approach is the view reference to the prod tables. you need to recreate the views in the cloned database. You can automate it with some scripts using information.

<li><b>Store backups or cold data in S3.</b>
Storing data in Snowflake that hasn’t been accessed by end users for an extended period is referred to as “cold data” in the data world and Creating backups of data in Snowflake may not be an optimal approach due to the associated cost of approx 40$ per TB on demand. Alternatively, data that is infrequently accessed or not accessed at all can be stored cost-effectively in cloud storage solutions like AWS S3. External tables can be created in Snowflake to access and utilize this data stored in S3.</li>

You can utilize Snowflake concepts known as STORAGE INTEGRATION and EXTERNAL STAGE to load and access data to and from cloud storage.

</ol>

Here are the queries we utilized to retrieve storage information from our Snowflake database.

`SELECT AVG(STORAGE_BYTES + STAGE_BYTES + FAILSAFE_BYTES) AVG_BYTES_PER_DAY,
       AVG_BYTES_PER_DAY/(1024*1024*1024*1024) AS TOTAL_STORAGE_IN_TB,
       TOTAL_STORAGE_IN_TB * 40 AS COST_IN_USD
FROM SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE;`

---

`SELECT SUM(COALESCE(ACTIVE_BYTES,0)) 
      + SUM(COALESCE(TIME_TRAVEL_BYTES,0)) 
      + SUM(COALESCE(TIME_TRAVEL_BYTES,0)) 
      + SUM(COALESCE(FAILSAFE_BYTES,0)) 
      + SUM(COALESCE(RETAINED_FOR_CLONE_BYTES,0)) AS BYTES,
        BYTES/(1024*1024*1024*1024) IN_TB,
        IN_TB*40 STORAGE_COST_IN_USD
 FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS 
 WHERE   DELETED = FALSE
 GROUP BY ALL;`

3. Transient table use case for temporary tables.

> _*As per snowflake documentation: Snowflake supports creating transient tables that persist until explicitly dropped and are available to all users with the appropriate privileges. Transient tables are similar to permanent tables with the key difference that they do not have a Fail-safe period. As a result, transient tables are specifically designed for transitory data that needs to be maintained beyond each session (in contrast to temporary tables), but does not need the same level of data protection and recovery provided by permanent tables.*_

In simpler terms, Transient tables function similarly to permanent tables but are more budget-friendly. However, keep in mind that once you drop these tables, data recovery is not possible.

In our use case, we are using them in the RAW/ DATA_SOURCE schema, the First point of landing data in the data warehouse.

You don’t need to explicitly specify TRANSIENT in the table DDL. Instead, you can create a TRANSIENT schema or database. All tables created in the TRANSIENT schema or database will be TRANSIENT in nature.

`CREATE TRANSIENT DATABASE AYUSH_RAW;
CREATE TRANSIENT SCHEMA AYUSH_PROD.RAW;`

4. Separate schema for temporary tables.
   In our data warehouse, we used to utilize temporary tables extensively, those were in all the schemas and databases making it challenging for us to monitor them effectively. we decided to centralize all temporary tables within a dedicated schema so that we can keep an eye on their storage efficiently.

5. Try to bring data from the external source system to S3.

In today’s cloud environment, a majority of source systems are hosted on the cloud, making it easy to efficiently extract and load data into cloud storage, which is generally more cost-effective than snowflake storage. Once the data is stored in the cloud, you can effortlessly transfer it from cloud storage to Snowflake using STORAGE_INTEGRATION and EXTERNAL STAGE.

This can help you to save a lot of money on the Storage and Compute side as well.

## Cost saving on the Compute side

In this section, I will describe the changes we implemented as part of resource management. Snowflake charges you per minute

Change the large or extra large warehouse size to small or x-small in non-peak hours.

Visualization tools such as Looker and Tableau require larger warehouses to meet reporting demands. However, during non-peak hours, we often observe random queries being executed by these visualization tools. It’s advisable to decrease the warehouse size during these off-peak periods. In our scenario, we initially utilized an X-Large warehouse for Looker. By scaling it down to an X-Small warehouse during nighttime, we managed to reduce costs by $12,000 annually.

You can utilize the ALTER command to modify the warehouse size and automate this process using your preferred orchestration tool. In our scenario, we have set up a directed acyclic graph (DAG) to run in the evening for downgrading the warehouse size and in the morning for upgrading it.

The below commands are for your reference.

`CREATE WAREHOUSE AYUSH_TEST
WITH
WAREHOUSE_TYPE = STANDARD
WAREHOUSE_SIZE = SMALL
AUTO_SUSPEND = 60
INITIALLY_SUSPENDED = TRUE;`

`ALTER WAREHOUSE AYUSH_TEST SET WAREHOUSE_SIZE = XSMALL;`

2. Add auto suspend to 60 seconds in each warehouse.

Always configure the property to automatically suspend the warehouse after 60 seconds of inactivity. This ensures that if a warehouse is not being used, it will be automatically suspended within a minute. FYI 60 seconds is the least value that you can set in the AUTO_SUSPEND property of the warehouse. You can always use ALTER WAREHOUSE to modify any property of the Snowflake warehouse.

3. Assign default warehouse to each role (prefer to assign x-small)

Assign users to roles and always designate an X-SMALL warehouse as the default for these roles. By doing this, users are required to use the X-SMALL warehouse until higher warehouses are necessary.

4. Add resource monitoring at the warehouse level and account level.

Resource monitoring in Snowflake allows for the monitoring of Snowflake credits at both the account and individual warehouse levels. You can immediately get notified if credits are overused at the account level or the warehouse level afterwards you can take the appropriate actions.

In our use case, we implemented resource monitoring at both the account level and at each warehouse level. This approach helps us identify issues and areas for improvement.

From the Snowsight UI and with the Simple CREATE [ OR REPLACE ] RESOURCE MONITOR command you can easily create the resource monitors. Below is an example of the creation of RESOURCE MONITOR at the warehouse level.

`CREATE RESOURCE MONITOR AYUSH_RS
CREDIT_QUOTA = 100 
FREQUENCY = 'MONTHLY' 
START_TIMESTAMP = 'IMMEDIATELY' 
TRIGGERS ON 80 PERCENT DO SUSPEND 
         ON 90 PERCENT DO SUSPEND_IMMEDIATE 
         ON 70 PERCENT DO NOTIFY;`

`ALTER WAREHOUSE AYUSH_TEST
SET RESOURCE_MONITOR = 'AYUSH_RS';`

5. Persist the data for the complex views.

Typically, we make the data accessible to reporting tools through views. However, at times, these views become overly intricate for querying efficiently. Both Tableau dashboards and Looker often utilize these views, consuming a significant amount of resources. Given that Snowflake operates as a database service, extended query times translate to increased costs.

In our use case, we either created materialized views or persisted data in the tables. This was done to enable reporting tools to efficiently and rapidly scan the data from both tables and materialized views.

We implemented several other additional steps to minimize and optimize Snowflake costs. few of these measures involved adopting incremental data loading and processing, we focused on implementing proper Role-Based Access Control (RBAC), optimizing queries, creating temporary tables for complex transformation logic processing, and educating users on Snowflake, emphasizing its differences from traditional databases.

By implementing the aforementioned changes, we successfully reduced the cost by 50% without compromising performance.

## Summary:

Snowflake’s flexibility and scalability have made it a popular choice for modern data warehousing needs. By applying these cost-saving strategies, you can maximize the value of Snowflake while keeping your cloud data warehousing expenses under control. Efficient data organization, query optimization, resource management, and data governance all play crucial roles in optimizing costs on Snowflake without compromising on performance.

Remember, continuous monitoring and fine-tuning are essential for maintaining cost efficiency over time. Regularly assess your Snowflake usage and adjust your practices as needed to ensure you’re getting the most out of your investment.

1. Selecting the Right Snowflake Instance:
   Choosing the appropriate Snowflake instance is crucial for cost savings. Evaluate the features and pricing of Standard, Enterprise, and Business Critical plans. Consider your specific requirements and conduct thorough research to make an informed decision.

2. Strategic Cloud Provider and Region Selection:
   Selecting the right cloud provider and region is essential to minimize data transfer costs. Analyze where other services in your company are hosted and align your Snowflake instance accordingly. For example, if your applications are on AWS, consider hosting your Snowflake instance on AWS.

3. Cost-Efficient Data Storage:
   Optimizing data storage can lead to significant cost savings. Implement strategies like cloning environments, utilizing external storage solutions like AWS S3 for cold data, and utilizing transient tables for temporary data.

4. Effective Resource Management:
   Managing resources efficiently can reduce costs on the compute side. Consider resizing warehouses during non-peak hours, setting auto-suspend to 60 seconds, assigning default warehouses to roles, and implementing resource monitoring at both account and warehouse levels.

5. Streamlining Complex Views and Queries:
   Optimize views and queries to enhance performance and reduce resource consumption. Consider using materialized views or persisting data in tables for reporting tools to efficiently access data.

6. Additional Measures for Cost Optimization:
   Implement incremental data loading and processing, adopt proper Role-Based Access Control (RBAC), optimize queries, create temporary tables for complex transformations, and provide user education on Snowflake.

Conclusion:
Snowflake's scalability and flexibility make it a preferred choice for modern data warehousing. By applying these cost-saving strategies, administrators can maximize the value of Snowflake while keeping expenses in check. Continuous monitoring and fine-tuning are crucial for long-term cost efficiency. Regularly assess usage patterns and adjust practices to ensure optimal return on investment.
