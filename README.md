# databricks_ABAC_pollciy_udf_hash_salt
We implement ABAC by applying a hashing policy to governed tags and using Databricks-managed secrets to hash sensitive data stored in Delta tables.
UDF for data masking 



This document presents the results of tests using custom UDFs in policies with Databricks-managed secrets for hashing and salting. We create a Databricks-managed secret scope that stores the OpenSSL‑generated keys for the hash and salt as two separate key–value pairs. These keys are then used in the policies to mask a column. Finally, we test the consistency of a join operation that uses this masked column as the join key.




Databricks secret scope

2 Key values were generated on terminal using openssl : < this can be any generator >

openssl rand -base64 32  —> hash
openssl rand -base64 32  —> salt



databricks secrets create-scope bne-vin-pii-masking 



Add key for hash : 



databricks secrets put-secret --json '{
  "scope": "bne-vin-pii-masking",
  "key": "masking_hash_salt",
  "string_value": "1N63cLisUEV**************”
}'



Add key for Salt

databricks secrets put-secret --json '{
  "scope": "bne-vin-pii-masking",
  "key": "masking_salt",
  "string_value": "fBjrBnDj5eizf**”*****************************
}'





Governed Tag 

This is created for the customer ID column ( as an example) 







Policy 

Add a policy at the catalog level applicable to all column with the tag “bne_va_customer_id : id” :






Policy function 

CREATE FUNCTION preprod001.default.mask_with_hash_plus_salt(
value STRING)
RETURNS STRING
COMMENT 'ABAC utility: Mask string values with asterisks unless user is in bne_va_admin group'
RETURN CASE
  WHEN is_account_group_member('bne-vin-admin') THEN value
  WHEN value IS NULL THEN NULL
  ELSE sha2(
           concat(
             -- 1) “hash” secret 
             secret('bne-vin-pii-masking', 'masking_hash_salt'),

             -- 2) normalized input value
             trim(lower(value)),

             -- 3) “salt” secret 
             secret('bne-vin-pii-masking', 'masking_salt')
           ),
           256
         )
END







Customers Table 
Let’s check the customers table and the assigned tags










Bookings Table 





Join 

A simple Join query on the customer_id (masked column) produces valid aggregations 


