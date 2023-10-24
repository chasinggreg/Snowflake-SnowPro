CREATE OR REPLACE TABLE EMPLOYEES(EMPLOYEE_ID INTEGER AUTOINCREMENT START = 1 INCREMENT = 1,
                       EMPLOYEE_NAME VARCHAR DEFAULT 'KASHISH',
                       LOAD_TIME DATE);


-- Stored procedure that INSERTS data TO a table
-- The INSERT statement in the stored procedure COPIES data to EMPLOYEES table
create or replace procedure load_employees_data(TODAY_DATE varchar)
  returns string not null
  language javascript
  as
  $$
    var sql_command = 'INSERT INTO EMPLOYEES(LOAD_TIME) VALUES(:1);'
    snowflake.execute(
        { 
        sqlText: sql_command, 
        binds: [TODAY_DATE] 
        }
        ); 
  return "SUCCEEDED"; 
  $$;

-- Task that calls the stored procedure every minute
create or replace task employees_load_task
  warehouse = COMPUTE_WH
  schedule = '1 minute'
as
  call load_employees_data(CURRENT_TIMESTAMP);
  

ALTER TASK employees_load_task RESUME;
ALTER TASK employees_load_task SUSPEND;



desc task employees_load_task;



TRUNCATE TABLE EMPLOYEES;

SHOW TASKS;