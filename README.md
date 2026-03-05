# Applying Databricks ABAC policies using Hashing for masking sensitive data
We implement ABAC by applying a hashing policy to governed tags and using Databricks-managed secrets to hash sensitive data stored in Delta tables.
UDF for data masking 


This guide presents the results of tests using custom UDFs in policies with Databricks-managed secrets for hashing and salting. We create a Databricks-managed secret scope that stores the OpenSSL‑generated keys for the hash and salt as two separate key–value pairs. These keys are then used in the policies to mask a column. Finally, we test the consistency of a join operation that uses this masked column as the join key.


## Databricks Managed secret 

2 Key values were generated on terminal using openssl : < this can be any generator >

On a terminal generate the 2 keys : 

`openssl rand -base64 32  —> hash`

`openssl rand -base64 32  —> salt`

### Login to databricks : 

`databricks auth login`

Create databricks managed secret scope : 

`databricks secrets create-scope bne-vin-pii-masking`

`databricks secrets list-scopes`

### Add key for hash : 

`databricks secrets put-secret --json '{
  "scope": "bne-vin-pii-masking",
  "key": "masking_hash_salt",
  "string_value": "1N63cLisUEV**************”
}' `

### Add key for Salt

`databricks secrets put-secret --json '{
  "scope": "bne-vin-pii-masking",
  "key": "masking_salt",
  "string_value": "fBjrBnDj5eizf**”*****************************
}'`


## Governed Tag 

This is created for the customer ID column ( as an example) 

<img width="874" height="481" alt="image" src="https://github.com/user-attachments/assets/5f2d361b-8896-44a7-879e-3cb937003333" />


## Policy 

Add a policy at the catalog level applicable to all column with the tag “bne_va_customer_id : id” :

<img width="704" height="622" alt="image" src="https://github.com/user-attachments/assets/6a0850ce-394a-4db2-8c70-1fea4d8b41e0" />

<img width="974" height="712" alt="image" src="https://github.com/user-attachments/assets/894899c3-d0ca-4bd3-8af4-21c7503ebd9c" />


### sample policy function 

```sql

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
```


### Customers Table

Let’s check the customers table and the assigned tags

<img width="963" height="688" alt="image" src="https://github.com/user-attachments/assets/a860b558-4659-4c30-97a8-96d601093bff" />


<img width="974" height="515" alt="image" src="https://github.com/user-attachments/assets/380420b3-4002-49de-9c1a-c8054e7ed607" />



### Bookings Table 

<img width="985" height="637" alt="image" src="https://github.com/user-attachments/assets/c01c923a-13fe-4457-9fcb-9e015ced06a2" />


<img width="989" height="630" alt="image" src="https://github.com/user-attachments/assets/a7913b2b-f49e-4319-9575-75729192e3c0" />


### Join 

A simple Join query on the customer_id (masked column) produces valid aggregations 

<img width="1033" height="672" alt="image" src="https://github.com/user-attachments/assets/4dd9382b-81ba-484d-8b75-b8c3587ab7dc" />

