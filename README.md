# SQL-Server-Scripts
My repository of SQL Server related scripts I've put together and tested in my own lab environment for various DBA tasks.
Resource Governor Notes
Okay, let's implement SQL Server Resource Governor to create a dedicated resource pool and workload group for potentially problematic queries. This configuration aims to limit their CPU, memory, and potentially I/O usage without crippling the entire server.
Assumptions:
You have CONTROL SERVER permissions to configure Resource Governor.
You can identify the logins, applications, or hosts that are likely sources of these resource-intensive queries.
This configuration provides a starting point. Thorough testing in a non-production environment is crucial before implementing in production. Resource limits might need significant tuning based on your specific workload and hardware performance.
Server Specs: 20 CPUs, 128 GB Memory (Using 128GB as it's a more standard power-of-2 value close to 126GB).
Strategy:
Create a Resource Pool (LimitedQueryPool): This pool will have defined maximums for CPU and Memory.
Create a Workload Group (LimitedQueryGroup): This group will define specific query behaviors (like memory grant limits per query, max degree of parallelism) and will be assigned to the LimitedQueryPool.
Create a Classifier Function: This User-Defined Function (UDF) will run at login time and assign connections matching specific criteria (e.g., login name, application name) to the LimitedQueryGroup. All other connections will go to the default group/pool.
Enable Resource Governor: Activate the configuration.

Explanation and Customization Points:
Pool LimitedQueryPool:
MAX_CPU_PERCENT = 20: Limits sessions in this pool to a maximum of 20% of the total server CPU capacity when the server CPU is under contention. If the server CPU is idle, they might exceed this briefly unless CAP_CPU_PERCENT is set. 20% on 20 cores means roughly 4 cores equivalent capacity maximum under load.
MAX_MEMORY_PERCENT = 15: Limits the total memory that can be granted to all queries running in this pool concurrently to 15% of the server's memory available for query grants (which is itself a portion of the total server memory).
CAP_CPU_PERCENT = 25: (Optional) A hard ceiling. Even if the rest of the server is idle, this pool won't use more than 25% CPU. Generally, rely on MAX_CPU_PERCENT first.
MIN_..._PERCENT = 0: This pool gets no guaranteed resources if other pools (like default) are demanding them.
IO Limits: MIN/MAX_IOPS_PER_VOLUME can limit disk I/O. This requires knowing your storage performance baseline (use tools like diskspd or Perfmon) and is often tuned later after observing behavior. Setting MAX_IOPS_PER_VOLUME = 0 means unlimited.
Group LimitedQueryGroup:
IMPORTANCE = Low: If CPU is scarce, requests from Medium (default) or High importance groups will be scheduled first.
REQUEST_MAX_MEMORY_GRANT_PERCENT = 25: This is critical. A single query in this group cannot ask for more than 25% of the pool's memory limit (which is 15% of total server memory). So, max grant per query = 25% of 15% = ~3.75% of total server grantable memory. This prevents one query from hogging all the pool's memory.
REQUEST_MEMORY_GRANT_TIMEOUT_SEC = 30: If a query needs memory but can't get it within 30 seconds (because the pool limit is reached), it will time out with an error (8645), preventing it from waiting indefinitely.
MAX_DOP = 4: Limits parallelism. Reducing DOP is often very effective at controlling CPU and memory for large scans/joins. Adjust 4 based on testing (1 or 2 are common for highly restricted groups).
Classifier Function udf_ResourceGovernorClassifier:
THIS IS THE MOST IMPORTANT PART TO CUSTOMIZE. You must edit the IF conditions to correctly identify the connections you want to limit. Use SUSER_SNAME(), APP_NAME(), HOST_NAME(), or combinations.
Ensure it always returns a valid group name (default if no specific condition is met).
Make sure you don't accidentally route system processes into your limited group.
Enabling and Monitoring:
ALTER RESOURCE GOVERNOR DISABLE/RECONFIGURE applies the changes. New logins after the reconfigure will use the classifier.
Use the provided DMV queries and Perfmon counters frequently after implementation to see if the limits are being hit, if queries are being queued or timed out, and if the overall server performance has improved for non-limited users. Adjust the pool/group limits as needed.
Remember to test this configuration extensively before deploying it to your production environment. Start with slightly less restrictive limits and tighten them gradually based on monitoring results.
