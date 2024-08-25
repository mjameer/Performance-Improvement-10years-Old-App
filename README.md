## Performance Improvement done for a 10years Old App

### Problem 

The referenced application has frequent performance issues and has been going down every day for about 17 straight days, due to which the application was not in a position to handle more than 5 to 10 concurrent users, which would lead to performance issues every day, even for a simple report generation.

### Solution  

I, upon investigating, identified multiple points of failure that are causing the application to run into performance issues which eventually bring the application down.

I have helped in applying a wave of performance tuning, automation, and modernization approaches to this application, which resulted in a 100% incident reduction, ranging from 23+ incidents per month to zero incidents.

![image](https://github.com/mjameer/Performance-Improvement-10years-Old-App/assets/11364104/ed2e9c9d-1546-4d7f-b828-70dc99abae1a)

#### Point of failure. 

-	The Database Connection is not closed at the code level. 
-	Application servers and database servers were running in different data centers. 
-	Minimal connection pool size. 
-	SQL queries run over 20 minutes during performance issues which would cause cascading disasters and consume all of the application's heap. 
-	Batch job keeps inserting into DB tables, which were indexed and have a size of over 400,000 records 
-	Existing batch jobs are scheduled in a window where they will continue running during peak business hours, causing DB deadlocks.

#### Fixes applied

-	Apply database connection closure syntax and migrate the application and database to the same data centers to avoid any network lags in traction. 
-	Configure the connection pool size from 10 to 150.
-	Set a query timeout for **select** queries that take longer than 5 minutes to run.
-	Ran the code against the sonar analyzer, Nexus, and Whitehat Scanner to identify and close any gaps associated with the code and any vulnerabilities associated with the library and fix them.
-	Worked with businesses in defining the retention policy to delete all data older than 8 years. which cleaned up the data size by 60%.
-	Worked with the business to redefine the batch job schedule time so that it runs before the business time window. For example, instead of running the batch job at 11 PM PST, we triggered the batch job at 4 PM PST, the business end time, giving the batch job about 7 additional hours to process.
-	Optimized Sequential Code: Converted sequential processing logic to parallel processing using Java 8 CompletableFuture and the Executor framework, improving execution speed and responsiveness.
-	Leveraged Java 8 Features: Refactored legacy code to utilize Java 8 parallel streams, which led to more efficient data handling and reduced processing time.
-	Applied Advanced Data Structures: Integrated data structures like Queues and Heaps where applicable, enhancing algorithmic efficiency and reducing time complexity.
-	These optimizations collectively resulted in a [10-20]% increase in processing performance and a [20-30]% reduction in execution time and latency, significantly boosting the scalability and responsiveness of the application.
-	Finally, we adopted a legacy application modernization approach that integrated various processes ranging from maximizing the application to implementing CI/CD processes using Jenkins, unit testing, TRO scanning(via Nexus, Whitehat DATS, and SATS), Sonar, and Artifactory integration and established Git as a proper version control management tool.

#### Technologies Used: 

- Java 8, Heap Dump analyzer, SQL query analyzer, Sonar, Nexus, Whitehat Maven, Jenkins, JUnit, Artifactory, Git

 ### Timeline

 The above solutions were done in three waves 

 - The first step was to stop the bleeding, so I applied the Db connection closure, Query timeout, and increase in the connection pool immediately. 
 - The next step was to work with users and DBA to apply and automate the retention policy, and also parallelly work with the interface team to change the Batch run file upload time and redefine the batch run schedule time.
 - Java 8 and DSA improvements
 - Finally, the rest of the modernization was done.  
 
