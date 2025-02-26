# **Web Server Log Analysis with Hive**  

**Project Overview**  
This project focuses on analyzing web server logs using **Apache Hive**. The goal is to extract insights such as:  
- **Website traffic trends**  
- **Most visited pages**  
- **Status code distribution**  
- **User agent analysis**  
- **Failed request patterns**  

The dataset is in CSV format, stored in HDFS, and queried using Hive to generate meaningful reports.  



**Implementation Approach**  

**1. Setting Up the Hive Environment**  

**Creating the Database**  

CREATE DATABASE web_logs;
USE web_logs;


**Creating an External Table**  

CREATE EXTERNAL TABLE web_server_logs (  
    ip STRING,  
    timestamp_ STRING,  
    url STRING,  
    status INT,  
    user_agent STRING  
)  
ROW FORMAT DELIMITED  
FIELDS TERMINATED BY ','  
STORED AS TEXTFILE  
LOCATION '/user/hive/warehouse/web_logs/';


**Loading Data into Hive Table**  

LOAD DATA INPATH '/user/hive/warehouse/web_logs/web_server_logs.csv' INTO TABLE web_server_logs;


**Creating a Partitioned Table for Efficient Queries**  

CREATE TABLE web_server_logs_partitioned (  
    ip STRING,  
    timestamp_ STRING,  
    url STRING,  
    user_agent STRING  
)  
PARTITIONED BY (status INT)  
ROW FORMAT DELIMITED  
FIELDS TERMINATED BY ','  
STORED AS TEXTFILE;


**Enabling Dynamic Partitioning**  

SET hive.exec.dynamic.partition.mode = nonstrict;
SET hive.exec.dynamic.partition = true;



**2. Data Processing & Analysis Queries**  

**Total Web Requests**  

SELECT COUNT(*) AS total_requests FROM web_server_logs;


**Status Code Distribution**  

SELECT status, COUNT(*) AS count FROM web_server_logs GROUP BY status;


**Top 3 Most Visited Pages**  

SELECT url, COUNT(*) AS visits FROM web_server_logs GROUP BY url ORDER BY visits DESC LIMIT 3;


**Traffic Trends Over Time (Minute-Level)**  

SELECT SUBSTR(timestamp_, 1, 16) AS time_slot, COUNT(*) AS request_count  
FROM web_server_logs  
GROUP BY SUBSTR(timestamp_, 1, 16)  
ORDER BY time_slot;


**Suspicious IPs with More than 3 Failed Requests (404, 500 Errors)**  

SELECT ip, COUNT(*) AS failed_requests  
FROM web_server_logs  
WHERE status IN (404, 500)  
GROUP BY ip  
HAVING COUNT(*) > 3;


**User Agent (Browser) Analysis**  

SELECT user_agent, COUNT(*) AS count FROM web_server_logs GROUP BY user_agent ORDER BY count DESC;




**3. Exporting Query Results**  

**Saving Query Results to HDFS**  

INSERT OVERWRITE DIRECTORY '/user/hadoop/web_logs/output'  
ROW FORMAT DELIMITED  
FIELDS TERMINATED BY ','  
SELECT * FROM web_server_logs;


**Saving Results to a Different Directory in HDFS**  

INSERT OVERWRITE DIRECTORY '/user/hive/output/web_logs_analysis'  
SELECT * FROM web_server_logs;


**Checking the Output in HDFS**  

hdfs dfs -ls /user/hadoop/web_logs/output


**Copying Results from HDFS to Docker Container**  

hdfs dfs -get /user/hadoop/web_logs/output /opt/output


**Copying Results from Docker to Local Machine**  

docker cp namenode:/opt/output output/output



**Execution Steps**  

1. **Copying the CSV file to ResourceManager and NameNode**  
   
   docker cp web_server_logs.csv resourcemanager:/opt/hadoop-3.2.1/share/hadoop/mapreduce/
   docker cp web_server_logs.csv namenode:/tmp/
   

2. **Logging into the NameNode Container**  
   
   docker exec -it namenode /bin/bash
   

3. **Creating an HDFS Directory for Web Logs**  
  
   hdfs dfs -mkdir -p /user/hive/warehouse/web_logs
   

4. **Moving the CSV File to HDFS**  
  
   hdfs dfs -put /tmp/web_server_logs.csv /user/hive/warehouse/web_logs/
  



**Challenges Faced and Resolutions**  

**Challenge**                    **Resolution** 

**Hive Table Creation Failed (`timestamp` is a Reserved Keyword)** | Renamed column `timestamp` to `timestamp_` to avoid conflicts. |
**Dynamic Partitioning Not Working** | Enabled partitioning using `SET hive.exec.dynamic.partition.mode = nonstrict;` |
**HDFS Copy Failure (`No such file or directory`)** | Verified the path using `hdfs dfs -ls /user/hadoop/web_logs/output` before running `hdfs dfs -get`. |
**Docker Copy Failing (`Could not find file`)** | Ensured the correct path inside the container (`namenode:/opt/output`) before executing `docker cp`. |
**Permission Denied while Copying Output to Local** | Used `sudo` while creating directories and copying files. |
**Output Directory Not Created Automatically in Docker** | Manually created the directory using `mkdir -p output/output` before copying. 

**Sample Input and Expected Output**  

**Sample Input (`web_server_logs.csv`)**  

192.168.1.1,2024-02-01 10:15,/home,200,Mozilla/5.0  
192.168.1.2,2024-02-01 10:16,/products,200,Chrome/90.0  
192.168.1.3,2024-02-01 10:17,/checkout,500,Safari/13.1  
192.168.1.10,2024-02-01 10:18,/home,404,Mozilla/5.0  
```

**Expected Output**  

**Total Web Requests:**  

Total Requests: 4


**Status Code Distribution:**  

200: 2  
404: 1  
500: 1  


**Top 3 Most Visited Pages:**  

/home: 2  
/products: 1  
/checkout: 1  


**User Agent Analysis:**  

Mozilla/5.0: 2  
Chrome/90.0: 1  
Safari/13.1: 1  


**Suspicious IPs (More than 3 failed requests):**  

(No suspicious IPs in sample data)




**Conclusion**  
This project successfully demonstrates how Hive can be used for analyzing large-scale **web server logs** stored in **HDFS**. Using partitioning and **optimized queries**, meaningful insights were extracted, such as the most visited pages, status code distributions, and suspicious IP addresses. The results were efficiently exported and transferred from HDFS to the local system using **Docker**.



