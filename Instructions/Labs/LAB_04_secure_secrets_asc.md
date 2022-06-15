---
lab:
    title: 'Lab: Secure application secrets using Key Vault'
    module: 'Module 4: Secure application secrets using Key Vaults'
---

# Challenge: Secure application secrets using Key Vault
# Student guide

## Challenge scenario

Your team is now running a first version of the spring-petclinic microservice application in Azure. However you are concerned that your application secrets are stored directly in configuration code. As a matter of fact, GitHub has been generating notifications informing you about this vulnerability. You want to remediate this issue and implement a secure method of storing application secrets that are part of the database connection string. In this unit, you will step through implementing such method. 

## Objectives

After you complete this challenge, you will be able to:

- Create an Azure Key Vault instance
- Store your connection string elements as Azure Key Vault secrets
- Create a managed identity for your microservices
- Grant the managed identity permissions to access the Azure Key Vault secrets
- Update application config
- Update, rebuild, and redeploy each app

## Lab Duration

- **Estimated Time**: 60 minutes

## Instructions

During this challenge, you'll:

- Create an Azure Key Vault instance
- Store your connection string elements as Azure Key Vault secrets
- Create a managed identity for your microservices
- Grant the managed identity permissions to access the Azure Key Vault secrets
- Update application config
- Update, rebuild, and redeploy each app

   > **Note**: The instructions provided in this exercise assume that you successfully completed the previous exercise and are using the same lab environment, including your Git Bash session with the relevant environment variables already set.

### Create an Azure Key Vault instance

You will start by creating an Azure Key Vault instance that will host your application secrets. You can use the following guidance to perform this task.

[Create Key Vault](https://docs.microsoft.com/en-us/azure/spring-cloud/tutorial-managed-identities-key-vault#set-up-your-key-vault)

<details>
<summary>hint</summary>
<br/>

1. From the Git Bash prompt, run the following command to create an Azure Key Vault instance. Note that the name of the service should be globally unique, so adjust it accordingly in case the randomly generated name is already in use. Keep in mind that the name can contain only lowercase letters, numbers and hyphens. The `$LOCATION` and `$RESOURCE_GROUP` variables contain the name of the Azure region and the resource group into which you deployed the Azure Spring Apps service in the previous exercise of this lab.

   ```bash
   KEYVAULT_NAME=springcloudkv$RANDOM
   az keyvault create \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --location $LOCATION \
       --sku standard
   ```

   > **Note**: Wait for the provisioning to complete. This might take about 2 minutes.

</details>

### Store your connection string elements as Azure Key Vault secrets

Now that your Key Vault provisioning is completed, you need to add to it a secret containing the connection string to the database hosted by Azure Database for MySQL Single Server. You can use the following guidance to perform this task.Tthese secrets should be called SPRING-DATASOURCE-USERNAME and SPRING-DATASOURCE-PASSWORD.

[Add a secret to Key Vault](https://docs.microsoft.com/en-us/azure/spring-cloud/tutorial-managed-identities-key-vault#set-up-your-key-vault)

<details>
<summary>hint</summary>
<br/>

1. Add the username and password of the Azure Database for MySQL Single Server admin account as secrets to your Key Vault by running the following commands from the Git Bash prompt:

   ```bash
   az keyvault secret set \
       --name SPRING-DATASOURCE-USERNAME \
       --value myadmin@$SQL_SERVER_NAME \
       --vault-name $KEYVAULT_NAME

   az keyvault secret set \
       --name SPRING-DATASOURCE-PASSWORD \
       --value $SQL_ADMIN_PASSWORD \
       --vault-name $KEYVAULT_NAME
   ```

</details>

### Create a managed identity for your microservices

The apps deployed as the Spring Petclinic microservices will connect to the newly created Key Vault using a managed identity. The process of creating a managed identity will automatically create an Azure Active Directory service principal for your application. Managed identities minimize the overhead associated with managing service principals, since their secrets used for authentication are automatically rotated. You can use the following guidance to determine how to assign a managed identity to a Spring Apps service application.

[Assign a Managed Identity](https://docs.microsoft.com/en-us/azure/spring-cloud/how-to-enable-system-assigned-managed-identity?tabs=azure-cli&pivots=sc-standard-tier#add-a-system-assigned-identity)

The following three apps of your application use the database hosted by the Azure Database for MySQL Single Server instance, so they will need to be assigned a managed identity:

- spring-petclinic-customers-service
- spring-petclinic-vets-service
- spring-petclinic-visits-service

<details>
<summary>hint</summary>
<br/>

1. Assign an identity to each of the three apps by running the following commands from Git Bash shell:

   ```bash
   az spring-cloud app identity assign \
       --service $SPRING_CLOUD_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --name customers-service \
       --system-assigned

   az spring-cloud app identity assign \
       --service $SPRING_CLOUD_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --name visits-service \
       --system-assigned

   az spring-cloud app identity assign \
       --service $SPRING_CLOUD_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --name vets-service \
       --system-assigned
   ```

1. Export the identity details to a separate environment variable for each of the apps so you can reuse it in the next part of the lab.

   ```bash
   CUSTOMERS_SERVICE_ID=$(az spring-cloud app identity show \
       --service $SPRING_CLOUD_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --name customers-service \
       --output tsv \
       --query principalId)

   VETS_SERVICE_ID=$(az spring-cloud app identity show \
       --service $SPRING_CLOUD_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --name vets-service \
       --output tsv \
       --query principalId)

   VISITS_SERVICE_ID=$(az spring-cloud app identity show \
       --service $SPRING_CLOUD_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --name visits-service \
       --output tsv \
       --query principalId)
   ```

</details>

### Grant the managed identity permissions to access the Azure Key Vault secrets

By now, you have created a managed identity for the spring-petclinic-customers-service, spring-petclinic-vets-service and spring-petclinic-visits-service. In this step, you need to grant these 3 managed identities access to the secrets your added to the Azure Key Vault instance. To accomplish this, you can use the following the guidance.

[Grant your app access to Key Vault](https://docs.microsoft.com/en-us/azure/spring-cloud/tutorial-managed-identities-key-vault#grant-your-app-access-to-key-vault)

The following three apps of your application use the database hosted by the Azure Database for MySQL Single Server instance, so their managed instances will need to be granted permissions to access the secrets:

- spring-petclinic-customers-service
- spring-petclinic-vets-service
- spring-petclinic-visits-service

<details>
<summary>hint</summary>
<br/>

1. Grant the get and list secrets permissions in the Azure Key Vault instance to each Spring Apps application managed identity by using Azure Key Vault access policy:

   ```bash
   az keyvault set-policy \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --secret-permissions get list  \
       --object-id $CUSTOMERS_SERVICE_ID

   az keyvault set-policy \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --secret-permissions get list  \
       --object-id $VETS_SERVICE_ID

   az keyvault set-policy \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --secret-permissions get list  \
       --object-id $VISITS_SERVICE_ID
   ```

</details>

### Update application config

You now have all relevant components in place to switch to the secrets stored in Azure Key Vault and remove them from your config repo. To complete your configuration, you now need to set the config repository to reference the Azure Key Vault instance. You also need to update the **pom.xml** file to ensure that the visits, vets and customers services use the **com.azure.spring:spring-cloud-azure-starter-keyvault-secrets** dependency. You can use the following guidance to accomplish this task.

[Azure Key Vault Secrets Spring Boot starter client library for Java](https://github.com/Azure/azure-sdk-for-java/blob/main/sdk/spring/azure-spring-boot-starter-keyvault-secrets/README.md)

<details>
<summary>hint</summary>
<br/>

1. From the Git Bash window, in the config repository you cloned locally, use your favorite text editor to open the application.yml file. Remove the lines 83 and 84 that contain the values of the admin user account name and its password for target datasource endpoint. 

   > **Note**: The lines 83 and 84 should have the following content (where the <your-server-name> and <myadmin-password> represent the name of the Azure Database for MySQL Single Server instance and the password you assigned to the myadmin account during its provisioning, respectively):

   ```yaml
    username: myadmin@<your-server-name>
    password: <myadmin-password>
   ```

1. Save the changes and push the updates you made to the **application.yml** file to your private GitHub repo by running the following commands from the Git Bash prompt:

   ```bash
   git add .
   git commit -m 'removed azure mysql credentials'
   git push
   ```

1. From the Git Bash window, in the config repository you cloned locally, use your favorite text editor to open again the application.yml file and append the following lines to it (where the `<key-vault-name>` placeholder represents the name of the Azure Key Vault you provisioned earlier in this exercise):

   ```yaml
   azure:
     keyvault:
       enabled: true
         property-source-enabled: true
         property-sources:
               - name: key-vault-property-source-1
              endpoint: https://springcloudlab3-kv.vault.azure.net/
              credential.managed-identity-enabled: true
   ```

1. Commit and push these changes to your remote config repository.

   ```bash
   git add .
   git commit -m 'added key vault'
   git push
   ```

### Update, rebuild, and redeploy each app

1. From the Git Bash window, in the config repository you cloned locally, use your favorite text editor to open **pom.xml** files of the customers, visits and vets services (within the spring-petclinic-customers-service, spring-petclinic-visits-service, and spring-petclinic-vets-service directories). For each, add the following dependencies (within the **<dependencies> </dependencies>** section) and save the change .

   ```xml
           <dependency>
              <groupId>com.azure.spring</groupId>
              <artifactId>spring-cloud-azure-starter-keyvault-secrets</artifactId>
           </dependency>
   ```

1. From the Git Bash window, in the config repository you cloned locally, use your favorite text editor to open the pom.xml file in the root directory of the cloned repo. Add to the file a dependency to **com.azure.spring**. This should be added within the **<dependencies><dependencyManagement></dependencies></dependencyManagement>** section.

   ```xml
       <dependencies>
           <dependencyManagement>
               //... existing dependencies

               <dependency>
                   <groupId>com.azure.spring</groupId>
                   <artifactId>spring-cloud-azure-dependencies</artifactId>
                   <version>${version.spring.cloud.azure}</version>
                   <type>pom</type>
                   <scope>import</scope>
               </dependency>

           </dependencies>
       </dependencyManagement>
   ```

1. In the same file, add a property for the **azure.version**. This should be added within the **<properties></properties>** section.

   ```xml
   <version.spring.cloud.azure>4.2.0</version.spring.cloud.azure>
   ```

1. Save the changes to the **pom.xml** file and close it.

1. Rebuild the services by running the following command in the root directory of the application.

   ```bash
   cd ~/spring-petclinic-microservices/
   mvn clean package -DskipTests
   ```

1. Verify that the build succeeds by reviewing the output of the `mvn clean package -DskipTests` command, which should have the following format: 

   ```bash
   [INFO] Reactor Summary for spring-petclinic-microservices 2.6.3:
   [INFO]
   [INFO] spring-petclinic-microservices ..................... SUCCESS [  0.505 s]
   [INFO] spring-petclinic-admin-server ...................... SUCCESS [  4.302 s]
   [INFO] spring-petclinic-customers-service ................. SUCCESS [  5.900 s]
   [INFO] spring-petclinic-vets-service ...................... SUCCESS [  3.650 s]
   [INFO] spring-petclinic-visits-service .................... SUCCESS [  3.520 s]
   [INFO] spring-petclinic-config-server ..................... SUCCESS [  1.122 s]
   [INFO] spring-petclinic-discovery-server .................. SUCCESS [  1.416 s]
   [INFO] spring-petclinic-api-gateway ....................... SUCCESS [  7.646 s]
   [INFO] ------------------------------------------------------------------------
   [INFO] BUILD SUCCESS
   [INFO] ------------------------------------------------------------------------
   [INFO] Total time:  28.985 s
   [INFO] Finished at: 2022-05-15T02:17:43Z
   [INFO] ------------------------------------------------------------------------
   ```

1. Redeploy the customers, visits and vets services to their respective apps in your Spring Apps service by running the following commands:

   ```bash
   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name customers-service \
                              --runtime-version Java_8 \
                              --no-wait \
                              --artifact-path spring-petclinic-customers-service/target/spring-petclinic-customers-service-2.6.1.jar \
                              --env SPRING_PROFILES_ACTIVE=mysql

   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name visits-service \
                              --runtime-version Java_8 \
                              --no-wait \
                              --artifact-path spring-petclinic-visits-service/target/spring-petclinic-visits-service-2.6.1.jar \
                              --env SPRING_PROFILES_ACTIVE=mysql

   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                              --resource-group $RESOURCE_GROUP \
                              --name vets-service \
                              --runtime-version Java_8 \
                              --no-wait \
                              --artifact-path spring-petclinic-vets-service/target/spring-petclinic-vets-service-2.6.1.jar \
                              --env SPRING_PROFILES_ACTIVE=mysql
   ```

1. Retest your application through its public endpoint. Ensure that the application is functional, while the connection string secrets are retrieved from Azure Key Vault.

1. To verify that this is the case, in the Azure Portal, navigate to the page of the Azure Key Vault instance you provisioned. On the Overview page, select the **Monitoring** tab and review the graph representing requests for access to the vault's secrets.

</details>

#### Review

In this lab, you implemented a secure method of storing application secrets that are part of the database connection string of Azure Spring Apps applications.
