# Multi cluster shared-disk                                                                                   
The Snowflake Cloud Data Platform is described as having which of the following architectures?                
                                                                                                              
A. Shared-disk<br>B. Shared-nothing<br>C. Multi-cluster shared data<br>D. Serverless query engine             
                                                                                                              
<details>                                                                                                     
<summary><strong>✅ Answer : </strong></summary>                                                              
<strong>C</strong>                                                                                            
                                                                                                              
Snowflake employs a unique architecture called multi-cluster shared data. This means that while the           
underlying storage layer is shared and accessible by all compute resources, processing is handled by          
independent, dynamically scalable compute clusters. Option A, "shared-disk," is incorrect because in this     
architecture all compute resources access the same storage, often leading to concurrency issues. Option B,    
"shared-nothing," which uses distributed storage and compute, does not accurately represent Snowflake’s       
single, shared storage for all compute resources. Option D, "serverless query engine," is partially correct as
Snowflake offers serverless features, but it's not the core defining architecture.                            
Snowflake's architecture separates storage and compute, utilizing a centralized data repository that's        
accessible by multiple independent compute clusters (virtual warehouses). These virtual warehouses are the    
processing engines, allowing for concurrent and isolated workloads without resource contention. The           
separation enables flexible scaling of compute up or down without impacting storage. This separation allows   
for independent scaling of both storage and compute resources based on workload requirements. Therefore,      
option C, "multi-cluster shared data" correctly defines the core architectural principle of Snowflake.        
Further research and validation can be found in Snowflake's official documentation:                           
Snowflake Architecture: https://docs.snowflake.com/en/user-guide/intro-key-concepts.html#snowflakearchitecture
Virtual Warehouses: https://docs.snowflake.com/en/user-guide/warehouses.html                                  
Key Concepts: https://docs.snowflake.com/en/user-guide/intro-key-concepts.html                                
</details>                                                                                                    
                                                                                                              
                                                                                                              
