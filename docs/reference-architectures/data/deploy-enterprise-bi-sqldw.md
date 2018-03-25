---
title: Gain business insights from relational data
description: Gain business insights from relational data
author: alexbuckgit
ms.date: 03/21/2018
---

## Deploy the solution

A deployment for this reference architecture is available on [GitHub][ref-arch-repo-folder]. It deploys the following:

  * A Windows virtual machine to simulate an on-premises database server. It includes SQL Server 2017 and related tools.
  * An Azure storage account that provides Blob storage to hold data exported from the SQL Server database.
  * An Azure SQL Data Warehouse instance.
  * An Azure Analysis Services instance.

### Prerequisites

Perform these prequisite steps before deploying the reference architecture to your subscription.

1. Clone, fork, or download the zip file for the [Azure reference architectures][ref-arch-repo] GitHub repository.

2. Install the [Azure Building Blocks][azbb-wiki].

3. From a command prompt, bash prompt, or PowerShell prompt, login to your Azure account by using the command below and following the instructions.

  ```bash
  az login  
  ```

### Deploy the simulated on-premises server using azbb

First you'll deploy a virtual machine as a simulated on-premises server, which includes SQL Server 2017 and related tools. This step also loads the sample [Wide World Importers OLTP database](/sql/sample/world-wide-importers/wide-world-importers-oltp-database) into SQL Server.

1. Navigate to the `data\enterprise-bi-sqldw\onprem\templates` folder of the repository you downloaded in the prerequisites above.

2. In the `onprem.parameters.json` file, replace the values for `adminUsername` and `adminPassword`. Also change the values in the `SqlUserCredentials` section to match the user name and password. Note the `.\\` prefix in the userName property.
    
    ```bash
    "SqlUserCredentials": {
      "userName": ".\\username",
      "password": "password"
    }
    ```

3. Run `azbb` as shown below to deploy the on-premises server.

    ```bash
    azbb -s <subscription_id> -g <resource_group_name> -l <location> -p onprem.parameters.json --deploy
    ```

4. The deployment may take 20 to 30 minutes to complete, which includes running the [Desired State Configuration](/powershell/dsc/overview) (DSC) to install the tools and restore the database. Verify the deployment in the Azure portal by reviewing the resources in the resource group. You should see the `sql-vm1` virtual machine and its associated resources.

### Deploy the Azure resources

This step provisions Azure SQL Data Warehouse and Azure Analysis Services, along with a Storage account. If you want, you can run this step in parallel with the previous step.

1. Navigate to the `data\enterprise-bi-sqldw\azure\templates` folder of the repository you downloaded in the prerequisites above.

2. Run the following Azure CLI command to create a resource group, replacing the bracketed parameters specified. Note that you can deploy to a different resource group than you used for the on-premises server in the previous step. 

    ```bash
    az group create --name <resource_group_name> --location <location>  
    ```

3. Run the following Azure CLI command to deploy the Azure resources, replacing the bracketed parameters specified. The `storageAccountName` parameter must follow the [naming rules](../../best-practices/naming-conventions.md#naming-rules-and-restrictions) for Storage accounts. For the `analysisServerAdmin` parameter, use your Azure Active Directory user principal name (UPN).

    ```bash
    az group deployment create --resource-group <resource_group_name> --template-file azure-resources-deploy.json --parameters "dwServerName"="<server_name>" "dwAdminLogin"="<admin_username>" "dwAdminPassword"="<password>" "storageAccountName"="<storage_account_name>" "analysisServerName"="<analysis_server_name>" "analysisServerAdmin"="user@contoso.com"
    ```

4. Verify the deployment in the Azure portal by reviewing the resources in the resource group. You should see a storage account, Azure SQL Data Warehouse instance, and Analysis Services instance.

5. Use the Azure portal to get the access key for the storage account. Select the storage account to open it. Under **Settings**, select **Access keys**. Copy the primary key value. You will use it in the next step.

### Export the source data to Azure Blob storage 

In this step, you will run a PowerShell script that uses bcp to export the SQL database to flat files on the VM, and then uses AzCopy to copy those files into Azure Blob Storage.

1. Use Remote Desktop to connect to the simulated on-premises VM you created previously.

2. While logged into the VM, run the following commands from a PowerShell window.  

    ```powershell
    cd 'C:\SampleDataFiles\reference-architectures\data\enterprise_bi_sqldw\onprem'

    .\Load_SourceData_To_Blob.ps1 -File .\sql_scripts\db_objects.txt -Destination 'https://<storage_account_name>.blob.core.windows.net/wwi' -StorageAccountKey '<storage_account_key>'
    ```

    For the `Destination` parameter, replace `<storage_account_name>` with the name the Storage account that you created previously. For the `StorageAccountKey` parameter, use the access key for that Storage account.

3. In the Azure portal, verify that the source data was copied to Blob storage by navigating to the storage account, selecting the Blob service, and opening the `wwi` container. You should see a list of tables prefaced with `WorldWideImporters_Application_*`.

### Execute the data warehouse scripts

1. From your Remote Desktop session, launch SQL Server Management Studio (SSMS). 

2. Connect to SQL Data Warehouse

    - Server type: Database Engine
    - Server name: `<dwServerName>.database.windows.net`, where `<dwServerName>` is the name that you specified when you deployed the Azure resources. You can get this name from the Azure portl.
    - Authentication: SQL Server Authentication

    Use the credentials that you specified when you deployed the Azure resources (`dwAdminLogin` and `dwAdminPassword`.

2. Navigate to the `C:\SampleDataFiles\reference-architectures\data\enterprise_bi_sqldw\azure\sqldw_scripts` folder on the VM. You will execute the scripts in this folder in numerical order (`STEP_1` through `STEP_7`).

3. Select the `master` database in SSMS and open the `STEP_1` script. Change the value of the password in the following line, then execute the script.

    ```sql
    CREATE LOGIN LoaderRC20 WITH PASSWORD = '<change this value>';
    ```

4. Select the `wwi` database in SSMS. Open the `STEP_2` script and execute the script. If you get an error, make sure you are running the script against the `wwi` database and not `master`.

5. Open a new connection to SQL Data Warehouse, using the `LoaderRC20` user and the password indicated in the `STEP_1` script.

6. Using this connection, open the `STEP_3` script. Set the following values in the script:

    - SECRET: Use the access key for your storage account.
    - LOCATION: Use the name of the storage account as follows: `wasbs://wwi@<storage_account_name>.blob.core.windows.net`.

7. Using the same connection, execute scripts `STEP_4` through `STEP_7` sequentially. Verify that each script completes successfully before running the next.

In SMSS, you should see a set of `prd.*` tables in the `wwi` database. To verify that the data was generated, run the following query: 

```sql
SELECT TOP 10 * FROM prd.CityDimensions
```

### Build the Azure Analysis Services model

1. From the Windows start menu on the on-premises server, open **SQL Server Data Tools 2015**.

2. Select **File -> New -> Project -> Templates -> Business Intelligence -> Analysis Services -> Analysis Services Tabular Project**. Name your project and click **OK**.
3. In the Tabular model designer dialog box, select the **Integrated workspace** option and set the **Compatibility level** to `SQL Server 2017 / Azure Analysis Services (1400)`, then click **OK**.
4. In the Tabular Model Explorer window, right-click the project and select **Import from Data Source**.
5. Select **All -> Azure SQL Data Warehouse**, then click **Connect**.
6. Set the Server value to the fully qualified name of your Azure SQL Data Warehouse, and the Database value to `wwi`, then click **OK**.
7. In the next dialog box, choose **Database** authentication and enter your Azure SQL Data Warehouse admin user name and password, then click **OK**.
8. In the Navigator dialog box, select the checkboxes for **prd.CityDimensions**, **prd.DateDimensions**, and **prd.SalesFact**, then click **Load**. When processing is complete, click **Close**. You should now see a tabular view of the data.
9. In the Tabular Model Explorer window, right-click the project and select **Model View -> Diagram View**.  
    a. In the diagram view, drag the **[prd.SalesFact].[WWI City ID]** field to the **[prd.CityDimensions].[WWI City ID]** field to create a relationship.  
    b. Drag the **[prd.SalesFact].[Invoice Date Key]** field to the **[prd.DateDimensions].[Date]** field.  
    c. From the Visual Studio **File** menu, choose **Save All**.  
10. In the Solution Explorer window, right-click on the project and select **Properties**. Set the Server property to the URL of your Azure Analysis Services instance. This value is the **Server Name** property of your Analysis Services instance displayed in the Azure portal. Click **OK**.
11. In the Solution Explorer window, right-click on the project and select **Deploy**. Sign into Azure if prompted. When processing is complete, click **Close**.
12. In the Azure portal, view the details for your Azure Analysis Services instance. Verify that your model appears in the list of models.

### Analyze the data via Power BI Desktop

1. From the Windows start menu on the on-premises server, open **Power BI Desktop**.
2. In the **Visualizations** pane, select the **Stacked Bar Chart** icon. Resize the workspace to make it larger.
3. In the Home ribbon tab, select **Get Data -> Analysis Services**.
4. Enter the URL of your Analysis Services instance, then click **OK**. Sign into Azure if prompted.
5. In the Navigator window, expand the tabular project that you deployed, select the model that you created, and click **OK**.
6. In the Fields pane, expand **prd.CityDimensions**.
    a. Drag the **WWI City ID** field to below the Axis label.
    b. Drag the **City** field to below to below the Legend label.
7. In the Fields pane, expand **prd.SalesFact**, and drag the **Total Excluding Tax** field to below the Value label.
8. Under Visual Level Filters, select **WWI City ID**, set the **Filter Type** to `Top N`, and set **Show Items** to `Top 10`.
9. From **prd.SalesFact**, drag the **Total Excluding Tax** field to below the **By Value** label, and click **Apply Filter**.

## Next steps

- For more information about this reference architecture, visit our [GitHub repository][ref-arch-repo-folder].
- Learn about the [Azure Building Blocks][azbb-repo].

<!-- links -->

[azure-cli-2]: /azure/install-azure-cli
[azbb-repo]: https://github.com/mspnp/template-building-blocks
[azbb-wiki]: https://github.com/mspnp/template-building-blocks/wiki/Install-Azure-Building-Blocks
[github-folder]: https://github.com/mspnp/reference-architectures/tree/master/data/enterprise_bi_sqldw
[ref-arch-repo]: https://github.com/mspnp/reference-architectures
[ref-arch-repo-folder]: https://github.com/mspnp/reference-architectures/tree/master/data/enterprise_bi_sqldw
