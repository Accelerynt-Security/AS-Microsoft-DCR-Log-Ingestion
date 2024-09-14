# AS-Microsoft-DCR-Log-Ingestion

Author: Accelerynt

For any technical questions, please contact info@accelerynt.com  

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAccelerynt-Security%2FAS-Microsoft-DCR-Log-Ingestion%2Fmain%2Fazuredeploy.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAccelerynt-Security%2FAS-Microsoft-DCR-Log-Ingestion%2Fmain%2Fazuredeploy.json)       

This playbook is designed to run on a timed trigger and pull Microsoft Graph and Microsoft Office logs to Microsoft Sentinel using Data Collection Endpoints and Data Collection Rules. While Microsoft does have built in connectors for this, they do not support multitenant functionality. This playbook is intended for multitenant organizations struggling with this limitation. This playbook is configured to grab the following logs for a tenant of your choosing:
* [Microsoft Graph Sign-In Logs](https://learn.microsoft.com/en-us/graph/api/signin-get?view=graph-rest-1.0&tabs=http)
* [Microsoft Graph Audit Logs](https://learn.microsoft.com/en-us/graph/api/directoryaudit-get?view=graph-rest-1.0&tabs=http)
* [Microsoft Office Activity Logs](https://learn.microsoft.com/en-us/office/office-365-management-api/office-365-management-activity-api-reference). 
                                                                                                                                     
![DCRLogIngestion_Demo_1](Images/DCRLogIngestion_Demo_1.png)

![DCRLogIngestion_Demo_2](Images/DCRLogIngestion_Demo_2.png)


#
### Requirements
                                                                                                                                     
The following items are required under the template settings during deployment: 

* A Microsoft Azure Active Directory [app registration](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#create-an-app-registration) with admin consent granted for "**AuditLog.Read.All**" in the "**Activity.Feed.Read**" API
* An [App Registration Azure key vault secret](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#create-an-app-registration-azure-key-vault-secret) containing your app registration client secret
* Note your [workspace location](https://portal.azure.com/#browse/Microsoft.OperationalInsights%2Fworkspaces) for the tenant that will be recieving data, as this will need to be the same for Data Collection Rules and Endpoints created in the steps below
* A [Microsoft Azure Data Collection Endpoint](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#create-the-data-collection-endpoints) for each of the log sources
* A [Microsoft Azure Data Collection Rule](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#create-the-data-collection-rules) for each of the log sources
* An [Azure key vault secret](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#create-an-azure-key-vault-secret) containing your client secret for each of your Data Collection Endpoints

# 
### Setup

#### Create an App Registration

From the tenant you wish to send the Microsoft Graph and Office data from, navigate to the Microsoft Azure Active Directory app registration page: https://portal.azure.com/#view/Microsoft_AAD_RegisteredApps/ApplicationsListBlade

Click "**New registration**".

![DCRLogIngestion_App_Registration_1](Images/DCRLogIngestion_App_Registration_1.png)

Enter "**AS-Microsoft-DCR-Log-Ingestion**" for the name, and select "**Accounts in any organizational directory**" for "**Supported account types**. All else can be left as is. Click "**Register**"

![DCRLogIngestion_App_Registration_2](Images/DCRLogIngestion_App_Registration_2.png)

Once the app registration is created, you will be redirected to the "**Overview**" page. Under the "**Essentials**" section, take note of the "**Application (client) ID**", as this will be needed for deployment.

![DCRLogIngestion_App_Registration_3](Images/DCRLogIngestion_App_Registration_3.png)

Next, you will need to add permissions for the app registration to call the Microsoft Graph and Office 365 API endpoints. From the left menu blade, click "**API permissions**" under the "**Manage**" section. Then, click "**Add a permission**".

![DCRLogIngestion_App_Registration_4](Images/DCRLogIngestion_App_Registration_4.png)

From the "**Select an API**" pane, click the "**Microsoft APIs**" tab and select "**Microsoft Graph**".

![DCRLogIngestion_App_Registration_5](Images/DCRLogIngestion_App_Registration_5.png)

Click "**Application permissions**", then paste "**AuditLog.Read.All**" in the search bar. Click the option matching the search, then click "**Add permission**".

![DCRLogIngestion_App_Registration_6](Images/DCRLogIngestion_App_Registration_6.png)

This process will need to be repeated for the Office 365 API. Click "**Add a permission**" once again and from the "**Select an API**" pane, click the "**Microsoft APIs**" tab and select "**Office 365 Management APIs**".

![DCRLogIngestion_App_Registration_7](Images/DCRLogIngestion_App_Registration_7.png)

Click "**Application permissions**", then paste "**ActivityFeed.Read**" in the search bar. Click the option matching the search, then click "**Add permission**".

![DCRLogIngestion_App_Registration_8](Images/DCRLogIngestion_App_Registration_8.png)

Admin consent will be needed before your app registration can use the assigned permission. Click "**Grant admin consent for (name)**".

![DCRLogIngestion_App_Registration_9](Images/DCRLogIngestion_App_Registration_9.png)

Lastly, a client secret will need to be generated for the app registration. From the left menu blade, click "**Certificates & secrets**" under the "**Manage**" section. Then, click "**New client secret**".

![DCRLogIngestion_App_Registration_10](Images/DCRLogIngestion_App_Registration_10.png)

Enter a description and select the desired expiration date, then click "**Add**".

![DCRLogIngestion_App_Registration_11](Images/DCRLogIngestion_App_Registration_11.png)

Copy the value of the secret that is generated, as this will be needed for [Create an Azure Key Vault Secret](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#create-an-azure-key-vault-secret).

![DCRLogIngestion_App_Registration_12](Images/DCRLogIngestion_App_Registration_12.png)


#### Create an App Registration Azure Key Vault Secret

The secret from the previous step will need to be stored in the tenant you wish to send the data to, as this is where the logic app will be deployed. Navigate to the Azure key vaults page: https://portal.azure.com/#view/HubsExtension/BrowseResource/resourceType/Microsoft.KeyVault%2Fvaults

Navigate to an existing key vault or create a new one. From the key vault overview page, click the "**Secrets**" menu option, found under the "**Settings**" section. Click "**Generate/Import**".

![DCRLogIngestion_Key_Vault_1](Images/DCRLogIngestion_Key_Vault_1.png)

Choose a name for the secret, such as "**AS-Microsoft-DCR-Log-Ingestion--App-Registration-Client-Secret**", and enter the client secret copied in the [previous section](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#create-an-app-registration). All other settings can be left as is. Click "**Create**". 

![DCRLogIngestion_Key_Vault_2](Images/DCRLogIngestion_Key_Vault_2.png)

Once your secret has been added to the vault, navigate to the "**Access policies**" menu option. Leave this page open, as you will need to return to it once the playbook has been deployed. See [Granting Access to Azure Key Vault](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#granting-access-to-azure-key-vault).

![DCRLogIngestion_Key_Vault_3](Images/DCRLogIngestion_Key_Vault_3.png)


#### Create the Data Collection Endpoints

From the tenant you wish to send the data to, navigate to the Microsoft Data Collection Endpoints page: https://portal.azure.com/#browse/microsoft.insights%2Fdatacollectionendpoints

Click "**Create**".

![DCRLogIngestion_Data_Collection_Endpoint_1](Images/DCRLogIngestion_Data_Collection_Endpoint_1.png)

Enter "**SignInLogsDCE**" as the Endpoint Name and select the Subscription and Resource Group. These should match the Subscription and Resource Group of the playbook you will deploy later. Ensure the Region location matches that of your workspace.  Click "**Review + create**".

![DCRLogIngestion_Data_Collection_Endpoint_2](Images/DCRLogIngestion_Data_Collection_Endpoint_2.png)

Click "**Create**".

![DCRLogIngestion_Data_Collection_Endpoint_3](Images/DCRLogIngestion_Data_Collection_Endpoint_3.png)

Repeat this process for "**AuditLogsDCE**".

![DCRLogIngestion_Data_Collection_Endpoint_4](Images/DCRLogIngestion_Data_Collection_Endpoint_4.png)

Repeat this process for "**OfficeActivityLogsDCE**".

![DCRLogIngestion_Data_Collection_Endpoint_5](Images/DCRLogIngestion_Data_Collection_Endpoint_5.png)

#### Create the Data Collection Rules

From the tenant you wish to send the data to, navigate to the Microsoft Log Analytics Workspace page: https://portal.azure.com/#browse/Microsoft.OperationalInsights%2Fworkspaces

Select the desired workspace.

![DCRLogIngestion_Data_Collection_Rule_1](Images/DCRLogIngestion_Data_Collection_Rule_1.png)

From the selected workspace, navigate to "**Tables**" located under settings, click "**Create**" and select "**New custom log (DCR based)**".

![DCRLogIngestion_Data_Collection_Rule_2](Images/DCRLogIngestion_Data_Collection_Rule_2.png)

First, click "**Create a new Data Collection Rule**" beleow the Data Collection Rule field. Then enter "**SignInLogsDCR**" for the name in the window that appears on the right. Ensure the Subscription, Resource Group, and Region all look correct, then click "**Done**".

![DCRLogIngestion_Data_Collection_Rule_3](Images/DCRLogIngestion_Data_Collection_Rule_3.png)

Next enter "**AzureSignInLogs**" as the table name and select "**SignInLogsDCE**" from the drop down list. If this option is not populating, double check the region used for the Data Collection Endpoint created in the previous step. Click "**Next**".

![DCRLogIngestion_Data_Collection_Rule_4](Images/DCRLogIngestion_Data_Collection_Rule_4.png)

The next step will prompt you for a data sample.

![DCRLogIngestion_Data_Collection_Rule_5](Images/DCRLogIngestion_Data_Collection_Rule_5.png)

Upload the file content located at [Samples/SignInLogsSample.json](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion/blob/main/Samples/SignInLogsSample.json), then click "**Next**".

![DCRLogIngestion_Data_Collection_Rule_6](Images/DCRLogIngestion_Data_Collection_Rule_6.png)

Click "**Create**".

![DCRLogIngestion_Data_Collection_Rule_7](Images/DCRLogIngestion_Data_Collection_Rule_7.png)

This process will need to be repeated for "**AuditLogsDCR**". Enter "**AzureAuditLogs**" as the table name and select "**AuditLogsDCE**" from the drop down list.

![DCRLogIngestion_Data_Collection_Rule_8](Images/DCRLogIngestion_Data_Collection_Rule_8.png)

Upload the file content located at [Samples/AuditLogsSample.json](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion/blob/main/Samples/AuditLogsSample.json), then click "**Next**".

![DCRLogIngestion_Data_Collection_Rule_9](Images/DCRLogIngestion_Data_Collection_Rule_9.png)

Click "**Create**".

![DCRLogIngestion_Data_Collection_Rule_10](Images/DCRLogIngestion_Data_Collection_Rule_10.png)




#### Create Azure Key Vault Secrets

Navigate to the Azure key vaults page: https://portal.azure.com/#view/HubsExtension/BrowseResource/resourceType/Microsoft.KeyVault%2Fvaults

Navigate to an existing key vault or create a new one. From the key vault overview page, click the "**Secrets**" menu option, found under the "**Settings**" section. Click "**Generate/Import**".

![DCRLogIngestion_Key_Vault_1](Images/DCRLogIngestion_Key_Vault_1.png)

Choose a name for the secret, such as "**AS-Microsoft-DCR-Log-Ingestion--App-Registration-Client-Secret**", and enter the client secret copied in the [previous section](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#create-an-app-registration). All other settings can be left as is. Click "**Create**". 

![DCRLogIngestion_Key_Vault_2](Images/DCRLogIngestion_Key_Vault_2.png)

Once your secret has been added to the vault, navigate to the "**Access policies**" menu option. Leave this page open, as you will need to return to it once the playbook has been deployed. See [Granting Access to Azure Key Vault](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#granting-access-to-azure-key-vault).

![DCRLogIngestion_Key_Vault_3](Images/DCRLogIngestion_Key_Vault_3.png)


#
### Deployment

To configure and deploy this playbook:
 
Open your browser and ensure you are logged into your Microsoft Sentinel workspace. In a separate tab, open the link to our playbook on the Accelerynt Security GitHub repository:

https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAccelerynt-Security%2FAS-Microsoft-DCR-Log-Ingestion%2Fmain%2Fazuredeploy.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAccelerynt-Security%2FAS-Microsoft-DCR-Log-Ingestion%2Fmain%2Fazuredeploy.json)                                             

Click the "**Deploy to Azure**" button at the bottom and it will bring you to the custom deployment template.

In the **Project Details** section:

* Select the "**Subscription**" and "**Resource Group**" from the dropdown boxes you would like the playbook deployed to.  

In the **Instance Details** section:

* **Playbook Name**: This can be left as "**AS-Microsoft-DCR-Log-Ingestion**" or you may change it.

* **Client ID**: Enter the Application (client) ID of your app registration referenced in [Create an App Registration](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#create-an-app-registration).

* **Key Vault Name**: Enter the name of the key vault referenced in [Create an Azure Key Vault Secret](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#create-an-azure-key-vault-secret).

* **Key Vault Secret Name**: Enter the name of the key vault Secret created in [Create an Azure Key Vault Secret](https://github.com/Accelerynt-Security/AS-Microsoft-DCR-Log-Ingestion#create-an-azure-key-vault-secret).

Towards the bottom, click on "**Review + create**". 

![DCRLogIngestion_Deploy_1](Images/DCRLogIngestion_Deploy_1.png)

Once the resources have validated, click on "**Create**".

![DCRLogIngestion_Deploy_2](Images/DCRLogIngestion_Deploy_2.png)

The resources should take around a minute to deploy. Once the deployment is complete, you can expand the "**Deployment details**" section to view them.
Click the one corresponding to the Logic App.

![DCRLogIngestion_Deploy_3](Images/DCRLogIngestion_Deploy_3.png)


#
### Granting Access to Azure Key Vault

Before the Logic App can run successfully, the key vault connection created during deployment must be granted access to the key vault storing your app registration client secret.

From the key vault "**Access policies**" page, click "**Create**".

![DCRLogIngestion_Key_Vault_Access_1](Images/DCRLogIngestion_Key_Vault_Access_1.png)

Select the "**Get**" checkbox under "**Secret permissions**", then click "**Next**".

![DCRLogIngestion_Key_Vault_Access_2](Images/DCRLogIngestion_Key_Vault_Access_2.png)

Paste "**AS-Microsoft-DCR-Log-Ingestion**" into the principal search box and click the option that appears. If the app registration also appears, select the option that does **not** match the Application (client) ID of your app registration. Click "**Next**" towards the bottom of the page.

![DCRLogIngestion_Key_Vault_Access_3](Images/DCRLogIngestion_Key_Vault_Access_3.png)

Navigate to the "**Review + create**" section and click "**Create**".

![DCRLogIngestion_Key_Vault_Access_4](Images/DCRLogIngestion_Key_Vault_Access_4.png)
