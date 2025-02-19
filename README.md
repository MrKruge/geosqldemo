# GeoSQLDemo

This project, as presented at DataPopKorn, demonstrates how to set up cross-tenant geo-replication using Azure SQL Database. Follow the instructions below to create a geo-replicated database environment across two Azure tenants.

## Prerequisites

1. **Azure Accounts**: You will need two separate Azure tenants. If you don't have them, you can sign up for free.
2. **Databases**: You need an Azure SQL Server and a Database set up on the primary tenant. Use AdventureWorks or StackOverflow databases as sample databases if needed.
3. **SQL User**: Ensure you have the required administrative permissions to create and manage databases and logins.
4. **IP Setup**: You must have the same public IP attached to both FW of SQL Server which is the Public IP of your local connection. This can be removed after and the traffic goes over the backbone of MSFT. 

## Setup Steps

### Step 1: Primary Tenant Configuration

1. **Connect to the Master Database** on your Azure SQL Server in the primary tenant:

    ```sql
    USE MASTER;
    CREATE LOGIN geosqluser WITH PASSWORD = 'ComplexPassword01';
    ALTER SERVER ROLE dbmanager ADD MEMBER geosqluser;
    ```

2. **Retrieve the SID** for `geosqluser` and store it securely; you'll need it for the secondary tenant:

    ```sql
    SELECT sid
    FROM sys.sql_logins
    WHERE name = 'geosqluser';
    ```

    - Example SID: `0x01060000000000640000000000000000797F828634ED824696634A2FC41CCF07`

3. **Create User in the User Database**:

    ```sql
    USE <YourDatabaseName>; -- Replace with your database name
    CREATE USER geosqluser FOR LOGIN geosqluser;
    ALTER ROLE db_owner ADD MEMBER geosqluser;
    ```

### Step 2: Secondary Tenant Configuration

1. **Create Login and User in the Master Database** on the secondary tenant Azure SQL Server. Use the SID from the primary tenant:

    ```sql
    USE MASTER;
    CREATE LOGIN geosqluser WITH PASSWORD = 'ComplexPassword01', SID = 0x01060000000000640000000000000000797F828634ED824696634A2FC41CCF07;
    CREATE USER geosqluser FOR LOGIN geosqluser;
    ALTER SERVER ROLE dbmanager ADD MEMBER geosqluser;
    ```

### Step 3: Configure Geo-Replication

1. **Back to the Primary Tenant**, execute the following command to add a secondary on another SQL Server using the `geosqluser` account:

    ```sql
    ALTER DATABASE [YourDatabaseName] ADD SECONDARY ON SERVER [YourSecondaryServerName];
    ```

2. **Check Geo-Replication Status**:

    ```sql
    USE MASTER;
    SELECT * FROM sys.dm_operation_status ORDER BY start_time DESC;

    SELECT 
        start_date,
        link_guid,
        partner_server,
        partner_database,
        replication_state_desc,
        role_desc
    FROM sys.geo_replication_links;

    -- This query shows if there is a data lag
    SELECT * FROM sys.dm_geo_replication_link_status;
    ```
    
3. **On the User Database Level**:

    ```sql
    USE YourDatabaseName;
    SELECT * FROM sys.dm_database_replica_states;
    ```

## Notes

- Replace placeholders like `[YourDatabaseName]` and `[YourSecondaryServerName]` with the actual database and server names used in your setup.
- Ensure network permissions and firewall settings allow necessary traffic between the primary and secondary Azure SQL Servers.
- MSFT Docs https://learn.microsoft.com/en-us/azure/azure-sql/database/active-geo-replication-configure-portal?view=azuresql&tabs=portal

## Conclusion

Following these steps will allow you to successfully configure cross-tenant geo-replication in Azure SQL Database. Be sure to manage user permissions and data access securely across both tenants.
