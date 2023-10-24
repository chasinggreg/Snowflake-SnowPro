----------------- ALTER -----------------
-- alter the structure of the objects

------ ALTER WAREHOUSE
  -- Suspends or resumes a virtual warehouse, or aborts all queries (and other SQL statements) for a warehouse. 
  -- Can also be used to rename or set/unset the properties for a warehouse.
  
-- To suspend a warehouse
alter warehouse test suspend;

alter warehouse test resume;

-- if warehouse is in suspended state, we can resume it by running below statement
alter warehouse test resume if suspended;

-- abort all queries running in warehouse
alter warehouse compute_wh abort all queries;

-- rename warehouse name
alter warehouse compute_wh rename to computer_wh;
alter warehouse computer_wh rename to compute_wh;