# Storage Cost                                                                                                                                                      
Snowflake provides two mechanisms to reduce data storage costs for short-lived tables. These mechanisms are:                                                        
(Choose two.)                                                                                                                                                       
                                                                                                                                                                    
A. Temporary Tables<br>B. Transient Tables<br>C. Provisional Tables<br>D. Permanent Tables                                                                          
                                                                                                                                                                    
<details>                                                                                                                                                           
<summary><strong>✅ Answer : </strong></summary>                                                                                                                    
<strong>A, B</strong>                                                                                                                                               
                                                                                                                                                                    
The correct answer is A. Temporary Tables and B. Transient Tables. Snowflake offers these two distinct                                                              
table types designed specifically for short-term data storage, directly contributing to cost optimization by                                                        
reducing long-term storage expenditures. Temporary tables exist solely within a user session and are                                                                
automatically dropped at the session's end, meaning they consume storage only while the session is active.                                                          
This makes them ideal for intermediate results or data needed for a specific operation, where data                                                                  
persistence is not required. Transient tables, on the other hand, persist across sessions but are not subject to                                                    
Snowflake's time travel or fail-safe mechanisms. They are therefore cheaper than permanent tables as the                                                            
storage cost is lower because they do not store the additional snapshots needed for those features. These                                                           
table types are perfect for data that doesn't need to be recovered from point-in-time snapshots or require the                                                      
safeguards of fail-safe functionality. Option C, Provisional Tables, is not a standard table type in Snowflake.                                                     
Option D, Permanent tables, are designed for long-term storage and are not suited for reducing costs related                                                        
to short-lived data. By utilizing temporary or transient tables where appropriate, users can effectively manage                                                     
storage costs associated with ephemeral data processing tasks in Snowflake.                                                                                         
Authoritative Links for Further Research:                                                                                                                           
Snowflake Documentation on Table Types: https://docs.snowflake.com/en/sql-reference/ddl-table-types                                                                 
Snowflake Documentation on Temporary Tables: https://docs.snowflake.com/en/user-guide/tables-temptransient                                                          
Snowflake Documentation on Transient Tables: https://docs.snowflake.com/en/user-guide/tables-temptransient                                                          
</details>                                                                                                                                                          
                                                                                                                                                                    
                                                                                                                                                                    
---                                                                                                                                                                 
What storage cost is completely eliminated when a Snowflake table is defined as transient?                                                                          
                                                                                                                                                                    
A. Active<br>B. Fail-safe<br>C. Staged<br>D. Time Travel                                                                                                            
                                                                                                                                                                    
<details>                                                                                                                                                           
<summary><strong>✅ Answer : </strong></summary>                                                                                                                    
<strong>B</strong>                                                                                                                                                  
                                                                                                                                                                    
Snowflake's transient tables are designed to minimize storage costs by eliminating the fail-safe                                                                    
period. Fail-safe is a crucial data recovery mechanism in Snowflake, providing a 7-day (for                                                                         
standard accounts) window for recovering data after it has been dropped or corrupted. This                                                                          
recovery is achieved by maintaining data snapshots which consume storage. When a table is                                                                           
defined as transient, Snowflake does not maintain these fail-safe snapshots, immediately                                                                            
reclaiming the storage upon data removal. This means the storage cost associated with fail-safe,                                                                    
which is specifically tied to the data snapshots, is completely avoided. Time Travel, another data                                                                  
recovery feature, is still operational for transient tables, however its storage costs are not                                                                      
eliminated as Time Travel is still retained for a period of time as specified by the user. Active                                                                   
storage reflects the cost of data currently residing in the table, and staged storage is for data                                                                   
being loaded into Snowflake, both of which are not eliminated by creating a transient table.                                                                        
Therefore, only fail-safe storage costs are removed when using a transient table.                                                                                   
Further Reading:                                                                                                                                                    
Snowflake Documentation on Transient Tables: https://docs.snowflake.com/en/userguide/tables-transient-temporary                                                     
Snowflake Documentation on Fail-safe: https://docs.snowflake.com/en/user-guide/data-failsafe                                                                        
Snowflake Documentation on Time Travel: https://docs.snowflake.com/en/user-guide/data-timetravel                                                                    
</details>                                                                                                                                                          
                                                                                                                                                                    
                                                                                                                                                                    
---                                                                                                                                                                 
What factors impact storage costs in Snowflake? (Choose two.)                                                                                                       
                                                                                                                                                                    
A. The account type<br>B. The storage file format<br>C. The cloud region used by the account<br>D. The type of data being stored<br>E. The cloud platform being used
                                                                                                                                                                    
<details>                                                                                                                                                           
<summary><strong>✅ Answer : </strong></summary>                                                                                                                    
<strong>A, C</strong>                                                                                                                                               
                                                                                                                                                                    
The correct answer is A and C. Storage costs in Snowflake are primarily influenced by                                                                               
the volume of data stored and the duration it is held. However, specific factors within the                                                                         
Snowflake environment also contribute significantly to these expenses. Option A, the                                                                                
account type, directly impacts storage costs. Different Snowflake account editions                                                                                  
(Standard, Enterprise, Business Critical) come with varying pricing structures, including                                                                           
potential discounts on storage based on consumption commitments. Hence, the chosen                                                                                  
account type will affect the base rate applied to storage consumption. Option C, the                                                                                
cloud region used by the account, also impacts costs because Snowflake's pricing varies                                                                             
across different cloud providers (AWS, Azure, GCP) and regions. This variation is based                                                                             
on the costs Snowflake incurs from those cloud providers. The location of data storage                                                                              
directly affects the price due to regional differences in infrastructure costs.                                                                                     
Option B, the storage file format, does not directly impact Snowflake's storage costs.                                                                              
Snowflake stores data in an internal, optimized format regardless of the original input                                                                             
format. Option D, the type of data being stored, does not affect storage pricing either,                                                                            
Snowflake charges based on the amount of data, not the type (e.g., text, images,                                                                                    
numeric). Option E, the cloud platform being used, indirectly impacts pricing via regional                                                                          
differences (as mentioned in option C) but it is not a direct factor because Snowflake                                                                              
charges are independent of underlying cloud platform. Therefore, options A and C are                                                                                
the primary factors directly influencing storage costs.                                                                                                             
Further research on Snowflake storage costs can be found at:                                                                                                        
Snowflake Pricing Guide: Provides a comprehensive overview of Snowflake's pricing                                                                                   
model, including different account types and regional variations.                                                                                                   
Snowflake Storage Costs: Snowflake documentation outlining specifics on how storage                                                                                 
costs are calculated.                                                                                                                                               
Snowflake Regional Availability: Details regions where Snowflake is available which can                                                                             
help determine cost differences for various locations.                                                                                                              
</details>                                                                                                                                                          
                                                                                                                                                                    
                                                                                                                                                                    
