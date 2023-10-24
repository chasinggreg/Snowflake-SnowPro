-- Check Tasks History

---- Retrieve the 100 most recent task executions (completed, still running, or scheduled in the future)
select *
  from table(information_schema.task_history())
  order by scheduled_time;
  
---- Retrieve the execution history for tasks in the account within a specified 60 minute block of time within the past 7 days:
select *
  from table(information_schema.task_history(
    scheduled_time_range_start=>to_timestamp_ltz('2020-09-08 11:00:00.000 -0700'),
    scheduled_time_range_end=>to_timestamp_ltz('2020-09-08 12:00:00.000 -0700')));

---- Retrieve the 10 most recent executions of a specified task (completed, still running, or scheduled in the future) scheduled within the last hour
select *
  from table(information_schema.task_history(
    scheduled_time_range_start=>dateadd('hour',-1,current_timestamp()),
    result_limit => 10,
    task_name=>'employees_load_task'));
