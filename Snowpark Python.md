There are three big things you need to learn. Dataframe api, stored procedures, and user defined functions. For dataframe api I’d just learn spark. The snowpark api is nearly identical. There are many more resources for learning spark than snowpark.

The big difference in snowpark is user defined function definition and stored procedures.

All you really need to know is stored procedures run on one node you can think about these like orchestrators. They can run dataframe operations, regular python code with libraries etc. This one node tells other nodes what to do.

For user functions these are distributed to many nodes. Think about regular snowflake functions like “sum”. These functions get applied to columns of a snowpark dataframe. This means snowflake will distribute computation to the cluster. These can leverage custom python libraries and objects.
