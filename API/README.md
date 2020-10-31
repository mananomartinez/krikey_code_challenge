## Part 2: API Scaling 
Player state has been exposed as an API endpoint using the SQL query you created in part one and Nodejs.
There are millions of rows in game_items, transactions and locations. 

How would you scale calculating player state for 50,000 concurrent requests? 

Would you introduce any new technologies? 

### My assumptions:
  - scenario refers to both the query times as well as processing the data queried.
  - greatest concern is performance rather than cost. 
  - scenario refers to a spike in traffic, not in a normal traffic load.
  - services are running on VM or container but not K8s. 
  - processing happens asynchronously and it

### Basic Solution
  - Load Balancer
    - Configured with either a `LeastBandwith `or a custom CPU-load based method.
      - Specially if there are sessions and the sticky flag is set to ensure an instance handles all requests for a session.
      -  Other methods, like `RoundRobin`, might not help as much with heavy traffic.

    -  Auto-scaling
        - I would set up auto-scaling based on a performance criteria to spin up new instances instances exceed a certain threshold for a specified period of time. 
        -  These instances would be spun down once the traffic returns to normal.

    - Stored Procedures and Views
      - Rather than have costly operations in the service, such as JOINs, move these to stored procedures for the more intensive queries
      -   Adding views that combine the necessary data which the service regularly will retrieve could reduce the number of queries and complexity in code.

### New Technologies 
  - Cache
     - (If switching from SQL to NO-SQL is not an option)
     - Maintain a fast cache, such as ElasticSearch, to handle the READ operations for the more common queries.  
       - This is because the database could be bogged down if a lot of WRITE operations occur.
           - WRITE operations in transactional DBs tend to be less performant than READ operations.
     - Downsides of this approach
       - Data could potentially be out of sync at any point.
       - Maintaining two versions of the data.
       - Performance cost in populating and updating 
           - Initial population of the cache, 
           - Any WRITE operation for SQL would have to update the cache as well as the DB. 
    - Serverless
        - Deploy  service in a server-less container (such as AWS Lambda).
            - No bottle neck waiting for requests to complete before other requests are processed. 
        - Cost
            - Costs for spinning a server less container per request can add quickly. 
            - If traffic density has consistently stayed high, options should be weighted, such as  paying for dedicated  or using spot VMs.

How would you reduce how long it takes the item locations query from part one to run? If the query is exposed as an API endpoint, how would you scale it for 50,000 concurrent requests? 
  - Denormalizing data
	 - The table game_item_locations relies on foreign keys to reference  items, locations and users, to minimize duplication of data as much as possible.
	 - It would be possible to write the location and game_item values into the game_item_locations to stop from having to perform extra queries to obtain the data.
  - Stored Procedure
    - If denormalizing is not an option.
    - Create a stored procedure for executing the query and join all the pertinent information for each row requested to keep it outside from the service, which can be more .  

[continue to Part 3](../kubernetes/README.md)    