---
title: "Tutorial: Migrate Azure Database for MySQL - Single Server to Flexible Server online using DMS via the Azure portal"
titleSuffix: "Azure Database Migration Service"
description: "Learn to perform an online migration from Azure Database for MySQL - Single Server to Flexible Server by using Azure Database Migration Service."
author: "adig"
ms.author: "adig"
manager: "pariks"
ms.reviewer: ""
ms.date: 09/07/2022
ms.service: dms
ms.topic: tutorial
ms.custom: seo-lt-2019
---

# Tutorial: Migrate Azure Database for MySQL - Single Server to Flexible Server online using DMS via the Azure portal

> [!NOTE]
> This article contains references to the term *slave*, a term that Microsoft no longer uses. When the term is removed from the software, we'll remove it from this article.

You can migrate an instance of Azure Database for MySQL – Single Server to Azure Database for MySQL – Flexible Server by using Azure Database Migration Service (DMS), a fully managed service designed to enable seamless migrations from multiple database sources to Azure data platforms. In this tutorial, we’ll perform an online migration of a sample database from an Azure Database for MySQL single server to a MySQL flexible server (both running version 5.7) using a DMS migration activity. 

> [!NOTE]
> DMS online migration is now in Preview. DMS supports migration for MySQL versions - 5.7 and 8.0, and also supports migration from lower version MySQL servers (v5.7 and above) to higher versions. In addition, DMS supports cross-region, cross-resource group, and cross-subscription migrations, so you can select a different region, resource group, and subscription for the target server than that specified for your source server.

In this tutorial, you will learn how to:

> [!div class="checklist"]
>
> * Implement best practices for creating a flexible server for faster data loads using DMS.
> * Create and configure a target flexible server.
> * Create a DMS instance.
> * Create a MySQL migration project in DMS.
> * Migrate a MySQL schema using DMS.
> * Run the migration.
> * Monitor the migration.
> * Perform post-migration steps.
> * Implement best practices for performing a migration.

## Prerequisites

To complete this tutorial, you need to:

* Create or use an existing instance of Azure Database for MySQL – Single Server (the source server).
* To complete the replicate changes migration successfully, ensure that the following prerequisites are in place:
  * Use the MySQL command line tool of your choice to determine whether log_bin is enabled on the source server. The Binlog is not always turned on by default, so verify that it is enabled before starting the migration. To determine whether log_bin is enabled on the source server, run the command: SHOW VARIABLES LIKE 'log_bin’
  * Ensure that the user has “REPLICATION CLIENT” and “REPLICATION SLAVE” permission on source server for reading and applying the bin log.
  * If you're targeting a replicate changes migration, configure the binlog_expire_logs_seconds parameter on the source server to ensure that binlog files are not purged before the replica commits the changes. We recommend at least two days to start. After a successful cutover, the value can be reset.
* To complete a schema migration successfully, on the source server, the user performing the migration requires the following privileges:
  * “READ” privilege on the source database.
  * “SELECT” privilege for the ability to select objects from the database
  * If migrating views, user must have the “SHOW VIEW” privilege.
  * If migrating triggers, user must have the “TRIGGER” privilege.
  * If migrating routines (procedures and/or functions), the user must be named in the definer clause of the routine. Alternatively, based on version, the user must have the following privilege:
    * For 5.7, have “SELECT” access to the “mysql.proc” table.
    * For 8.0, have “SHOW_ROUTINE” privilege or have the “CREATE ROUTINE,” “ALTER ROUTINE,” or “EXECUTE” privilege granted at a scope that includes the routine.
  * If migrating events, the user must have the “EVENT” privilege for the database from which the event is to be shown.

## Limitations

As you prepare for the migration, be sure to consider the following limitations.

* When migrating non-table objects, DMS does not support renaming databases.
* When migrating to a target server with bin_log enabled, be sure to enable log_bin_trust_function_creators to allow for creation of routines and triggers.
* When migrating the schema, DMS does not support creating a database on the target server. 
* Currently, DMS does not support migrating the DEFINER clause for objects. All object types with definers on the source are dropped and after the migration the default definer for tables will be set to the login used to run the migration.
* Currently, DMS only supports migrating a schema as part of data movement. If nothing is selected for data movement, the schema migration will not occur. Note that selecting a table for schema migration also selects it for data movement.
* Online migration support is limited to the ROW binlog format.
* Online migration only replicates DML changes; replicating DDL changes is not supported. Do not make any schema changes to the source while replication is in progress.

## Best practices for creating a flexible server for faster data loads using DMS

DMS supports cross-region, cross-resource group, and cross-subscription migrations, so you are free to select appropriate region, resource group and subscription for your target flexible server. Before you create your target flexible server, consider the following configuration guidance to help ensure faster data loads using DMS.

* Select the compute size and compute tier for the target flexible server based on the source single server’s pricing tier and VCores as in the following table:

    | Single Server Pricing Tier | Single Server VCores | Flexible Server Compute Size | Flexible Server Compute Tier |
    | ------------- | ------------- |:-------------:|:-------------:|
    | Basic\* | 1 | General Purpose | Standard_D16ds_v4 |
    | Basic\* | 2 | General Purpose | Standard_D16ds_v4 |
    | General Purpose\* | 4 | General Purpose | Standard_D16ds_v4 |
    | General Purpose\* | 8 | General Purpose | Standard_D16ds_v4 |
    | General Purpose | 16 | General Purpose | Standard_D16ds_v4 |
    | General Purpose | 32 | General Purpose | Standard_D32ds_v4 |
    | General Purpose | 64 | General Purpose | Standard_D64ds_v4 |
    | Memory Optimized | 4 | Business Critical | Standard_E4ds_v4 |
    | Memory Optimized | 8 | Business Critical | Standard_E8ds_v4 |
    | Memory Optimized | 16 | Business Critical | Standard_E16ds_v4 |
    | Memory Optimized | 32 | Business Critical | Standard_E32ds_v4 |

\* For the migration, select General Purpose 16 VCores compute for the target flexible server for faster migrations. Scale back to the desired compute size for the target server after migration is complete by following the compute size recommendation in the Performing post-migration activities section later in this article.

* The MySQL version for the target flexible server must be greater than or equal to that of the source single server.
* Unless you need to deploy the target flexible server in a specific zone, set the value of the Availability Zone parameter to ‘No preference’.
* For network connectivity, on the Networking tab, if the source single server has private endpoints or private links configured, select Private Access; otherwise, select Public Access.
* Copy all firewall rules from the source single server to the target flexible server.
* Copy all the name/value tags from the single to flex server during creation itself.

## Create and configure the target flexible server

With these best practices in mind, create your target flexible server and then configure it.

* Create the target flexible server. For guided steps, see the quickstart [Create an Azure Database for MySQL flexible server](./../mysql/flexible-server/quickstart-create-server-portal.md).
* Next to configure the newly created target flexible server, proceed as follows: 
  * The user performing the migration requires the following permissions:
    * Ensure that the user has “REPLICATION_APPLIER” or “BINLOG_ADMIN” permission on target server for applying the bin log.
    * Ensure that the user has “REPLICATION SLAVE” permission on target server.
    * Ensure that the user has “REPLICATION CLIENT” and “REPLICATION SLAVE” permission on source server for reading and applying the bin log.
    * To create tables on the target, the user must have the “CREATE” privilege.
    * If migrating a table with “DATA DIRECTORY” or “INDEX DIRECTORY” partition options, the user must have the “FILE” privilege.
    * If migrating to a table with a “UNION” option, the user must have the “SELECT,” “UPDATE,” and “DELETE” privileges for the tables you map to a MERGE table.
    * If migrating views, you must have the “CREATE VIEW” privilege.
    Keep in mind that some privileges may be necessary depending on the contents of the views. Please refer to the MySQL docs specific to your version for “CREATE VIEW STATEMENT” for details
    * If migrating events, the user must have the “EVENT” privilege.
    * If migrating triggers, the user must have the “TRIGGER” privilege.
    * If migrating routines, the user must have the “CREATE ROUTINE” privilege.
  * Create a target database with the same name as that on source server, though it need not be populated with tables/views, etc.
    * Set the appropriate character, collations, and any other applicable schema settings prior to starting the migration, as this may affect the DEFAULT set in some of the object definitions.
    * Additionally, if migrating non-table objects, be sure to use the same name for the target schema as is used on the source.
  * Configure the server parameters on the target flexible server as follows:
    * Set the TLS version and require_secure_transport server parameter to match the values on the source server.
    * Configure server parameters on the target server to match any non-default values used on the source server.
    * To ensure faster data loads when using DMS, configure the following server parameters as described.
      * max_allowed_packet – set to 1073741824 (i.e., 1GB) to prevent any connection issues due to large rows.
      * slow_query_log – set to OFF to turn off the slow query log. This will eliminate the overhead caused by slow query logging during data loads.
      * innodb_buffer_pool_size – can only be increased by scaling up compute for Azure Database for MySQL server. Scale up the server to 64 vCore General Purpose SKU from the Pricing tier of the portal during migration to increase the innodb_buffer_pool_size.
      * innodb_io_capacity & innodb_io_capacity_max - Change to 9000 from the Server parameters in Azure portal to improve the IO utilization to optimize for migration speed.
      * innodb_write_io_threads - Change to 4 from the Server parameters in Azure portal to improve the speed of migration.
  * Configure the replicas on the target server to match those on the source server.
  * Replicate the following server management features from the source single server to the target flexible server:
    * Role assignments, Roles, Deny Assignments, classic administrators, Access Control (IAM)
    * Locks (read-only and delete)
    * Alerts
    * Tasks
    * Resource Health Alerts

## Set up DMS

With your target flexible server deployed and configured, you next need to set up DMS to migrate your single server to a flexible server.

### Register the resource provider

To register the Microsoft.DataMigration resource provider, perform the following steps.

1. Before creating your first DMS instance, sign in to the Azure portal, and then search for and select **Subscriptions**.
    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/1-subscriptions.png" alt-text="Screenshot of a Select subscriptions from Azure Marketplace.":::

2. Select the subscription that you want to use to create the DMS instance, and then select **Resource providers**.
    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/2-resource-provider.png" alt-text="Screenshot of a Select Resource Provider.":::

3. Search for the term “Migration”, and then, for **Microsoft.DataMigration**, select **Register**.
    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/3-register.png" alt-text="Screenshot of a Register your resource provider.":::

### Create a Database Migration Service (DMS) instance

01. In the Azure portal, select + **Create a resource**, search for the term “Azure Database Migration Service”, and then select **Azure Database Migration Service** from the drop-down list.
    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/4-dms-portal-marketplace.png" alt-text="Screenshot of a Search Azure Database Migration Service.":::

02. On the **Azure Database Migration Service** screen, select **Create**.
    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/5-dms-portal-marketplace-create.png" alt-text="Screenshot of a Create Azure Database Migration Service instance.":::
  
03. On the **Select migration scenario and Database Migration Service** page, under **Migration scenario**, select **Azure Database for MySQL-Single Server** as the source server type, and then select **Azure Database for MySQL** as target server type, and then select **Select**.
    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/6-create-dms-service-scenario-online.png" alt-text="Screenshot of a Select Migration Scenario.":::

04. On the **Create Migration Service** page, on the **Basics** tab, under **Project details**, select the appropriate subscription, and then select an existing resource group or create a new one.

05. Under **Instance details**, specify a name for the service, select a region, and then verify that **Azure** is selected as the service mode.

06. To the right of **Pricing tier**, select **Configure tier**.
    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/7-project-details.png" alt-text="Screenshot of a Select Configure Tier.":::

07. On the Configure page, select the pricing tier and number of vCores for your DMS instance, and then select Apply.
    For more information on DMS costs and pricing tiers, see the [pricing page](https://aka.ms/dms-pricing).
    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/8-configure-pricing-tier.png" alt-text="Screenshot of a Select Pricing tier.":::

    Next, we need to specify the VNet that will provide the DMS instance with access to the source single server and the target flexible server.

08. On the **Create Migration Service** page, select **Next : Networking >>**.

09. On the **Networking** tab, select an existing VNet from the list or provide the name of new VNet to create, and then select **Review + Create**.
    For more information, see the article [Create a virtual network using the Azure portal.](./../virtual-network/quick-create-portal.md).
    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/8-1-networking.png" alt-text="Screenshot of a Select Networking.":::

    > [!IMPORTANT]
    > Your vNet must be configured with access to both the source single server and the target flexible server, so be sure to:
    >
    > * Create a server-level firewall rule or [configure VNET service endpoints](./../mysql/single-server/how-to-manage-vnet-using-portal.md) for both the source and target Azure Database for MySQL servers to allow the VNet for Azure Database Migration Service access to the source and target databases.
    > * Ensure that your VNet Network Security Group (NSG) rules don't block the outbound port 443 of ServiceTag for ServiceBus, Storage, and Azure Monitor. For more details about VNet NSG traffic filtering, see [Filter network traffic with network security groups](./../virtual-network/virtual-network-vnet-plan-design-arm.md).
    
    > [!NOTE]
    > If you want to add tags to the service, first select Next : Tags to advance to the Tags tab first. Adding tags to the service is optional.

10. Navigate to the **Review + create** tab, review the configurations, view the terms, and then select **Create**.
     :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/9-review-create.png" alt-text="Screenshot of a Select Review+Create.":::
    Deployment of your instance of DMS now begins. The message **Deployment is in progress** appears for a few minutes, and then the message changes to **Your deployment is complete**.

11. Select **Go to resource**.
     :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/9-1-go-to-resource.png" alt-text="Screenshot of a Select Go to resource.":::

### Create a migration project

To create a migration project, perform the following steps.  

1. In the Azure portal, select **All services**, search for Azure Database Migration Service, and then select **Azure Database Migration Services**.

    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/10-dms-search.png" alt-text="Screenshot of a Locate all instances of Azure Database Migration Service.":::

2. In the search results, select the DMS instance that you just created, and then select + **New Migration Project**.

    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/11-select-create.png" alt-text="Screenshot of a Select a new migration project.":::

3. On the **New migration project** page, specify a name for the project, in the Source server type selection box, select **Azure Database For MySQL – Single Server**, in the Target server type selection box, select **Azure Database For MySQL**, in the **Migration activity type** selection box, select **Online migration**, and then select **Create and run activity**.
    > [!NOTE]
    > Selecting Create project only as the migration activity type will only create the migration project; you can then run the migration project at a later time.

    :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/12-create-project-online.png" alt-text="Screenshot of a Create a new migration project.":::

### Configure the migration project

To configure your DMS migration project, perform the following steps.

1. On the **Select source** screen, specify the connection details for the source MySQL instance.
       :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/select-source-online.png" alt-text="Screenshot of an Add source details screen.":::

2. Select **Next : Select target>>**, and then, on the **Select target** screen, specify the connection details for the target flexible server.
       :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/select-target-online.png" alt-text="Screenshot of a Select target.":::

3. Select **Next : Select databases>>**, and then, on the Select databases tab, under [Preview] Select server objects, select the server objects that you want to migrate.
       :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/16-select-db.png" alt-text="Screenshot of a Select database.":::

4. In the **Select databases** section, under **Source Database**, select the database(s) to migrate.
    The non-table objects in the database(s) you specified will be migrated, while the items you didn’t select will be skipped. You can only select the source and target databases whose names match that on the source and target server.
    If you select a database on the source server that doesn’t exist on the target database, you will see a warning message ‘Not available at Target’ and you won’t be able to select the database for migration.

5. Select **Next : Select databases>>** to navigate to the Select tables tab.
    Before the tab populates, DMS fetches the tables from the selected database(s) on the source and target and then determines whether the table exists and contains data.

6. Select the tables that you want to migrate.
    If the selected source table doesn't exist on the target server, the online migration process will ensure that the table schema and data is migrated to the target server.
   :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/17-select-tables.png" alt-text="Screenshot of a Select Tables.":::

    DMS validates your inputs, and if the validation passes, you will be able to start the migration.

7. After configuring for schema migration, select **Review and start migration**.
    > [!NOTE]
    > You only need to navigate to the Configure migration settings tab if you are trying to troubleshoot failing migrations.

8. On the **Summary** tab, in the **Activity name** text box, specify a name for the migration activity, and then review the summary to ensure that the source and target details match what you previously specified.
   :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/18-summary-online.png" alt-text="Screenshot of a Select Summary.":::

9. Select **Start migration**.
    The migration activity window appears, and the Status of the activity is Initializing. The Status changes to Running when the table migrations start.
   :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/running-online-migration.png" alt-text="Screenshot of a Running status.":::

### Monitor the migration

1. Once the **Initial Load** activity is completed, navigate to the **Initial Load** tab to view the completion status and the number of tables completed.
       :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/completed-initial-load-online.png" alt-text="Screenshot of a completed initial load migration.":::

2. Once the **Initial Load** activity is completed, you are navigated to the **Replicate Data Changes** tab automatically. You can monitor the migration progress as the screen is auto-refreshed every 30 seconds. Select **Refresh** to update the display and view the seconds behind source as and when needed.

     :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/running-replicate-data-changes.png" alt-text="Screenshot of a Monitoring migration.":::

3. Monitor the **Seconds behind source** and as soon as it nears 0, proceed to start cutover by clicking on the **Start Cutover** menu tab at the top of the migration activity screen. Follow the steps in the cutover window before you are ready to perform a cutover. Once all steps are completed, click on **Confirm** and next click on **Apply**.
     :::image type="content" source="media/tutorial-azure-mysql-single-to-flex-online/21-complete-cutover-online.png" alt-text="Screenshot of a Perform cutover.":::

## Perform post-migration activities

When the migration is complete, be sure to complete the following post-migration activities.

* Perform sanity testing of the application against the target database to certify the migration.
* Update the connection string to point to the new target flexible server.
* Delete the source single server after you have ensured application continuity.
* If you scaled-up the target flexible server for faster migration, scale it back by selecting the compute size and compute tier for the target flexible server based on the source single server’s pricing tier and VCores as in the table below.

    | Single Server Pricing Tier | Single Server VCores | Flexible Server Compute Size | Flexible Server Compute Tier |
    | ------------- | ------------- |:-------------:|:-------------:|
    | Basic | 1 | Burstable | Standard_B1s |
    | Basic | 2 | Burstable | Standard_B2s |
    | General Purpose | 4 | General Purpose | Standard_D4ds_v4 |
    | General Purpose | 8 | General Purpose | Standard_D8ds_v4 |
* Clean up Data Migration Service resources:
    1. In the Azure portal, select **All services**, search for Azure Database Migration Service, and then select **Azure Database Migration Services**.
    2. Select your migration service instance from the search results and select **Delete service**.
    3. On the confirmation dialog box, in the **TYPE THE DATABASE MIGRATION SERVICE NAME** textbox, specify the name of the service, and then select **Delete**.

## Migration best practices

When performing a migration, be sure to keep the following best practices in mind.

* As part of discovery and assessment, take the server SKU, CPU usage, storage, database sizes, and extensions usage as some of the critical data to help with migrations.
* Perform test migrations before migrating for production:
  * Test migrations are important for ensuring that you cover all aspects of the database migration, including application testing. The best practice is to begin by running a migration entirely for testing purposes. After a newly started migration enters the Replicate Data Changes phase with minimal lag, make your Flexible Server target the primary database server. Use that target for testing the application to ensure expected performance and results. If you're migrating to a higher MySQL version, test for application compatibility.
  * After testing is completed, you can migrate the production databases. At this point, you need to finalize the day and time of production migration. Ideally, there's low application use at this time. All stakeholders who need to be involved should be available and ready. The production migration requires close monitoring. For an online migration, the replication must be completed before you perform the cutover, to prevent data loss.
* Redirect all dependent applications to access the new primary database and open the applications for production usage.
* After the application starts running on the target flexible server, monitor the database performance closely to see if performance tuning is required.

## Next steps

* For information about Azure Database for MySQL - Flexible Server, see [Overview - Azure Database for MySQL Flexible Server](./../mysql/flexible-server/overview.md).
* For information about Azure Database Migration Service, see the article [What is Azure Database Migration Service?](./dms-overview.md).
* For information about known issues and limitations when performing migrations using DMS, see the article [Common issues - Azure Database Migration Service](./known-issues-troubleshooting-dms.md).
* For troubleshooting source database connectivity issues while using DMS, see the article [Issues connecting source databases](./known-issues-troubleshooting-dms-source-connectivity.md).
