---
lab:
    title: 'End-to-end security with Azure Synapse Analytics'
    module: 'Module 8'
---

# Lab 8 - End-to-end security with Azure Synapse Analytics

In this lab, you will learn how to secure a Synapse Analytics workspace and its supporting infrastructure. You will observe the SQL Active Directory Admin, manage IP firewall rules, manage secrets with Azure Key Vault and access those secrets through a Key Vault linked service and pipeline activities. You will understand how to implement column-level security, row-level security, and dynamic data masking when using dedicated SQL pools.

After completing this lab, you will be able to:

- Secure Azure Synapse Analytics supporting infrastructure
- Secure the Azure Synapse Analytics workspace and managed services
- Secure Azure Synapse Analytics workspace data

This lab will guide you through several  security-related steps that cover an end-to-end security story for Azure Synapse Analytics. Some key take-aways from this lab are:

1. Leverage Azure Key Vault to store sensitive connection information, such as access keys and passwords for linked services as well as in pipelines.

2. Introspect the data that is contained within the SQL Pools in the context of potential sensitive/confidential data disclosure. Identify the columns representing sensitive data, then secure them by adding column-level security. Determine at the table level what data should be hidden from specific groups of users then define security predicates to apply row level security (filters) on the table. If desired, you also have the option of applying Dynamic Data Masking to mask sensitive data returned in queries on a column by column basis.

## Lab setup and pre-requisites

Before starting this lab, you must complete at least the setup steps in **Lab 4: *Explore, transform, and load data into the Data Warehouse using Apache Spark***.

This lab uses the dedicated SQL pool you created in the previous lab. You should have paused the SQL pool at the end of the previous lab, so resume it by following these instructions:

1. Open Azure Synapse Studio (<https://web.azuresynapse.net/>).
2. Select the **Manage** hub.
3. Select **SQL pools** in the left-hand menu. If the **SQLPool01** dedicated SQL pool is paused, hover over its name and select **&#9655;**.

    ![The resume button is highlighted on the dedicated SQL pool.](images/resume-dedicated-sql-pool.png "Resume")

4. When prompted, select **Resume**. It will take a minute or two to resume the pool.
5. Continue to the next exercise while the dedicated SQL pool resumes.

> **Important:** Once started, a dedicated SQL pool consumes credits in your Azure subscription until it is paused. If you take a break from this lab, or decide not to complete it; follow the instructions at the end of the lab to pause your SQL pool!

## Exercise 1 - Securing Azure Synapse Analytics supporting infrastructure

Azure Synapse Analytics (ASA) is a powerful solution that handles security for many of the resources that it creates and manages. In order to run ASA, however, some foundational security measures need to be put in place to ensure the infrastructure that it relies upon is secure. In this exercise, we will walk through securing the supporting infrastructure of ASA.

### Task 1 - Observing the SQL Active Directory admin

 The SQL Active Directory Admin can be a user (the default) or group (best practice so that more than one user can be provided these permissions) security principal. The principal assigned to this will have administrative permissions to the SQL Pools contained in the workspace.

1. In the Azure Portal (<https://portal.azure.com>), browse to your lab resource group, and from the list of resources open your Synapse workspace (do not launch Synapse Studio).

2. On the left menu, select **Azure Active Directory** and observe who is listed as a SQL Active Directory Admin. Is it a user or group?

    ![On the SQL Active Directory Admin screen, SQL Active Directory admin is selected from the left menu, and the Active Directory Admin is highlighted.](images/lab5_workspacesqladadmin.png)

### Task 2 - Manage IP firewall rules

Having robust Internet security is a must for every technology system. One way to mitigate internet threat vectors is by reducing the number of public IP addresses that can access the Azure Synapse Analytics Workspace through the use of IP firewall rules. The Azure Synapse Analytics workspace will then delegate those same rules to all managed public endpoints of the workspace, including those for SQL pools and SQL Serverless endpoints.

1. In the Azure Portal,  on the blade for your Synapse workspace, select **Networking**.

2. Notice that an IP Firewall rule of **Allow All** has already been created for you in the lab environment. If you wanted to add your specific IP address you would instead select **+ Add Client IP** from the taskbar menu (you do not need to do this now).

    ![On the Synapse Workspace screen, the + Add Client IP button is selected from the toolbar menu.](images/lab5_synapsefirewalladdclientipmenu.png)

    > **Note**: When connecting to Synapse from your local network, certain ports need to be open. To support the functions of Synapse Studio, ensure outgoing TCP ports 80, 443, and 1433, and UDP port 53 are open.

## Exercise 2 - Securing the Azure Synapse Analytics workspace and managed services

### Task 1 - Managing secrets with Azure Key Vault

When dealing with connectivity to external data sources and services, sensitive connection information such as passwords and access keys should be properly handled. It is recommended that this type of information be stored in an Azure Key Vault. Leveraging Azure Key Vault not only protects against secrets being compromised, it also serves as a central source of truth; meaning that if a secret value needs to be updated (such as when cycling access keys on a storage account), it can be changed in one place and all services consuming this key will start pulling the new value immediately. Azure Key Vault encrypts and decrypts information transparently using 256-bit AES encryption, which is FIPS 140-2 compliant.

1. In the Azure Portal, open the resource group for this lab, and from the list of resources, select the **Key vault** resource.

    ![Key vault is selected in the resource group.](images/resource-group-key-vault.png "Key vault")

2. On the left menu, under Settings, select **Access Policies**.

3. Observe that Managed Service Identity (MSI) representing your Synapse workspace (it has a name similar to **asaworkspace*xxxxxxx***) has already been listed under Application and it has 4 selected Secret Management Operations.

    ![The Synapse workspace account and its assigned secret permissions are highlighted.](images/key-vault-access-policies.png "Access policies")

4. Select the drop-down that reads **4 selected** under **Secret Management Operations**, observe that **Get** (which allows your workspace to retrieve the values of secrets from Key Vault) and **List** (which allows your workspace to enumerate secrets) are set.

### Task 2 - Use Azure Key Vault for secrets when creating Linked Services

Linked Services are synonymous with connection strings in Azure Synapse Analytics. Azure Synapse Analytics linked services provides the ability to connect to nearly 100 different types of external services ranging from Azure Storage Accounts to Amazon S3 and more. When connecting to external services, having secrets related to connection information is almost guaranteed. The best place to store these secrets is the Azure Key Vault. Azure Synapse Analytics provides the ability to configure all linked service connections with values from Azure Key Vault.

In order to leverage Azure Key Vault in linked services, you must first add your key vault resource as a linked service in Azure Synapse Analytics.

1. In Azure Synapse Studio, select the **Manage** hub from the left menu.

    ![The Manage hub is selected.](images/manage-hub.png "Manage hub")

2. Beneath **External Connections**, select **Linked Services**, observe that a Linked Service pointing to your Key Vault has been created in the environment.

    ![The Key Vault linked service is highlighted.](images/key-vault-linked-service.png "Key Vault linked service")

Since we have the Azure Key Vault set up as a linked service, we can leverage it when defining new linked services. Every New linked service provides the option to retrieve secrets from Azure Key Vault. The form requests the selection of the Azure Key Vault linked service, the secret name, and (optional) specific version of the secret.

![A New linked service form is displayed with the Azure Key Vault setting highlighted with the fields described in the preceding paragraph.](images/lab5_newlinkedservicewithakv.png)

### Task 3 - Secure workspace pipeline runs

It is recommended to store any secrets that are part of your pipeline in Azure Key Vault. In this task you will retrieve these values using a Web activity, just to show the mechanics. The second part of this task demonstrates using a Web activity in the pipeline to retrieve a secret from the Key Vault.

1. Return to the Azure portal.

2. In the blade for your **asakeyvault*xxxxxxx*** Azure Key Vault resource, and select **Secrets** from the left menu. Then, in the top toolbar, select **+ Generate/Import**.

   ![In Azure Key Vault, Secrets is selected from the left menu, and + Generate/Import is selected from the top toolbar.](images/lab5_pipelinekeyvaultsecretmenu.png)

3. Create a secret, with the name `PipelineSecret` and assign it a value of `IsNotASecret`, and select the **Create** button.

   ![The Create a secret form is displayed populated with the specified values.](images/lab5_keyvaultcreatesecretforpipeline.png)

4. Open the secret that you just created, drill into the current version, and copy the value in the Secret Identifier field. Save this value in a text editor, or retain it in your clipboard for a future step.

    ![On the Secret Version form, the Copy icon is selected next to the Secret Identifier text field.](images/lab5_keyvaultsecretidentifier.png)

5. Switch back to Synapse Studio, then select the **Integrate** hub from the left menu.

    ![Integrate hub.](images/integrate-hub.png "Integrate hub")

6. On the **Integrate** pane, in the **+** menu, select **Pipeline**.

    ![On the Integrate blade the + button is expanded with the Pipeline item selected beneath it.](images/new-pipeline.png)

7. On the **Pipeline** tab, in the **Activities** pane search for **Web** and then drag an instance of a **Web** activity to the design area.

    ![In the Activities pane, Web is entered into the search field. Under General, the Web activity is displayed in the search results. An arrow indicates the drag and drop movement of the activity to the design surface of the pipeline. The Web activity is displayed on the design surface.](images/lab5_pipelinewebactivitynew.png)

8. Select the **Web1** web activity, and select the **Settings** tab. Fill out the form as follows:

    1. **URL**: Paste the Key Vault Secret Identifier value you copied in step 4 above, then **append** `?api-version=7.1` to to the end of this value. For example, it should look something like: `https://asakeyvaultNNNNN.vault.azure.net/secrets/PipelineSecret/f808d4fa99d84861872010f6c8d25c68?api-version=7.1`.
  
    2. **Method**: Select **Get**.

    3. For **Authentication** select **Managed Identity**. We have already established an Access Policy for the Managed Service Identity of our Synapse workspace, this means that the pipeline activity has permissions to access the key vault via an HTTP call.
  
    4. **Resource**: Enter **<https://vault.azure.net>**

        ![The Web Activity Settings tab is selected and the form is populated with the values indicated above.](images/lab5_pipelineconfigurewebactivity.png)

9. From the Activities pane, add a **Set variable** activity to the design surface of the pipeline.

    ![An arrow goes from the set variable item under Activities to the pipeline canvas.](images/pipeline-activities-set-variable.png "Activities: Set variable")

10. On the design surface of the pipeline, select the **Web1** activity and drag a **Success** activity pipeline connection (green box) to the **Set variable1** activity.

11. With the pipeline selected in the designer (e.g., neither of the activities are selected), select the **Variables** tab and add a new **String** parameter named **`SecretValue`**.

      ![The design surface of the pipeline is shown with a new pipeline arrow connecting the Web1 and Set variable1 activities. The pipeline is selected, and beneath the design surface, the Variables tab is selected with a variable with the name of SecretValue highlighted.](images/lab5_newpipelinevariable.png)

12. Select the **Set variable1** activity and select the **Variables** tab. Fill out the form as follows:

    1. **Name**: Select **SecretValue** (the variable that we just created).

    2. **Value**: Enter `@activity('Web1').output.value`

        ![On the pipeline designer, the Set Variable1 activity is selected. Below the designer, the Variables tab is selected with the form set the previously specified values.](images/lab5_pipelineconfigsetvaractivity.png)

13. Debug the pipeline by selecting **Debug** from the toolbar menu. When it runs observe the inputs and outputs of both activities from the **Output** tab of the pipeline.

    ![The pipeline toolbar is displayed with the Debug item highlighted.](images/lab5_pipelinedebugmenu.png)

    ![In the output of the pipeline, the Set variable 1 activity is selected with its input displayed. The input shows the value of NotASecret that was pulled from the key vault being assigned to the SecretValue pipeline variable.](images/lab5_pipelinesetvariableactivityinputresults.png)

    > **Note**: On the **Web1** activity, on the **General** tab there is a **Secure Output** checkbox that when checked will prevent the secret value from being logged in plain text, for instance in the pipeline run, you would see a masked value ***** instead of the actual value retrieved from the Key vault. Any activity that consumes this value should also have their **Secure Input** checkbox checked.

### Task 4 - Secure Azure Synapse Analytics dedicated SQL pools

Transparent Data Encryption (TDE) is a feature of SQL Server that provides encryption and decryption of data at rest, this includes: databases, log files, and back ups. When using this feature with Synapse Analytics dedicated SQL pools, it will use a built-in symmetric Database Encryption Key (DEK) that is provided by the pool itself. With TDE, all stored data is encrypted on disk, when the data is requested, TDE will decrypt this data at the page level as it's read into memory, and vice-versa encrypting in-memory data before it gets written back to disk. As with the name, this happens transparently without affecting any application code. When creating a dedicated SQL pool through Synapse Analytics, Transparent Data Encryption is not enabled. The first part of this task will show you how to enable this feature.

1. In the **Azure Portal**, open your resource group, then locate and open the **SqlPool01** dedicated SQL pool resource.

    ![The SQLPool01 resource is highlighted in the Resource Group.](images/resource-group-sqlpool01.png "Resource Group: SQLPool01")

2. On the **SQL pool** resource blade, select **Transparent data encryption** from the left-hand menu. **DO NOT** turn on data encryption.

   ![On the SQL pool resource screen, Transparent data encryption is selected from the menu.](images/tde-form.png)

    By default, this option is turned off. When you enable data encryption on this dedicated SQL pool, the pool is taken offline for a few minutes while TDE is applied.

## Exercise 3 - Securing Azure Synapse Analytics workspace data

### Task 1 - Column Level Security

It is important to identify data columns that hold sensitive information. Types of sensitive could be social security numbers, email addresses, credit card numbers, financial totals, and more. Azure Synapse Analytics allows you define permissions that prevent users or roles select privileges on specific columns.

1. In **Azure Synapse Studio**, in the **Develop** hub, expand the **SQL scripts** section, and select **Column Level Security**.
2. In the toolbar, connect to the **SQLPool01** database.
3. In the query window, **run each step individually** by highlighting the statement(s) in the step in the query window, and selecting the **Run** button from the toolbar (or press **F5**).
4. Close the script tab. If prompted select **Discard all changes**.

### Task 2 - Row level security

1. In the **Develop** hub, in the **SQL scripts** section, select **Row Level Security**.
2. In the toolbar, connect to the **SQLPool01** database.
3. In the query window, **run each step individually** by highlighting the statement(s) for the step in the query window, and selecting the **Run** button from the toolbar (or press **F5**).
4. Close the script tab. If prompted select **Discard all changes**.

### Task 3 - Dynamic data masking

1. In the **Develop** hub, in the **SQL scripts** section, select **Dynamic Data Masking**.
2. In the toolbar, connect to the **SQLPool01** database.
3. In the query window, **run each step individually** by highlighting the statement(s) for the step in the query window, and selecting the **Run** button from the toolbar (or press **F5**).
4. Close the script tab. If prompted select **Discard all changes**.

## Important: Pause your SQL pool

Complete these steps to free up resources you no longer need.

1. In Synapse Studio, select the **Manage** hub.
2. Select **SQL pools** in the left-hand menu. Hover over the **SQLPool01** dedicated SQL pool and select **||**.

    ![The pause button is highlighted on the dedicated SQL pool.](images/pause-dedicated-sql-pool.png "Pause")

3. When prompted, select **Pause**.

## Reference

- [IP Firewalls](https://docs.microsoft.com/azure/synapse-analytics/security/synapse-workspace-ip-firewall)
- [Synapse Workspace Managed Identity](https://docs.microsoft.com/azure/synapse-analytics/security/synapse-workspace-managed-identity)
- [Synapse Managed VNet](https://docs.microsoft.com/azure/synapse-analytics/security/synapse-workspace-managed-vnet)
- [Synapse Managed Private Endpoints](https://docs.microsoft.com/azure/synapse-analytics/security/synapse-workspace-managed-private-endpoints)
- [Secure your Synapse Workspace](https://docs.microsoft.com/azure/synapse-analytics/security/how-to-set-up-access-control)
- [Connect to your Synapse Workspace using private links](https://docs.microsoft.com/azure/synapse-analytics/security/how-to-connect-to-workspace-with-private-links)
- [Create a Managed private endpoint to your data source](https://docs.microsoft.com/azure/synapse-analytics/security/how-to-create-managed-private-endpoints)
- [Granting Permissions to Workspace Managed Identity](https://docs.microsoft.com/azure/synapse-analytics/security/how-to-grant-workspace-managed-identity-permissions)

## Other Resources

- [Managing access to workspaces, data and pipelines](https://docs.microsoft.com/azure/synapse-analytics/sql/access-control)
- [Analyze with Apache Spark](https://docs.microsoft.com/azure/synapse-analytics/get-started-analyze-spark)
- [Visualize data with Power BI](https://docs.microsoft.com/azure/synapse-analytics/get-started-visualize-power-bi)
- [Control storage account access for SQL on-demand](https://docs.microsoft.com/azure/synapse-analytics/sql/develop-storage-files-storage-access-control)
