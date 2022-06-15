---
lab:
    title: 'Lab: Migrate a Spring Apps application to Azure'
    module: 'Module 2: Migrate a Spring Apps application to Azure'
---

# Challenge: Migrate a Spring Apps application to Azure
# Student manual

## Challenge scenario

You have established a plan for migrating the Spring Petclinic application to Azure Spring Apps. It is now time to perform the actual migration of the Spring Petclinic application components.

## Objectives

After you complete this challenge, you will be able to:

- Create an Azure Spring Apps service
- Set up the config server
- Create an Azure MySQL Database service
- Deploy the Spring Petclinic app components to the Spring Apps service
- Provide a publicly available endpoint for the Spring Petclinic application
- Test the application through the publicly available endpoint

## Challenge Duration

- **Estimated Time**: 120 minutes

## Instructions

During this challenge, you'll:
- Create an Azure Spring Apps service
- Set up the config server
- Create an Azure MySQL Database service
- Deploy the Spring Petclinic app components to the Spring Apps service
- Provide a publicly available endpoint for the Spring Petclinic application
- Test the application through the publicly available endpoint

   > **Note**: Your workstation contains the following components installed:

   - Visual Studio Code available from [Visual Studio Code Downloads](https://code.visualstudio.com/download)
   - Git for Windows 2.3.61 available from [Git Downloads](https://git-scm.com/downloads)
   - [Apache Maven 3.8.5](apache-maven-3.8.5-bin.zip) available from [Apache Maven Project downloads](https://maven.apache.org/download.cgi)
   - Java Development Kit (JDK) available from [JDK downloads](https://download.oracle.com/java/18/latest/jdk-18_windows-x64_bin.msi)
   - jq available from [jq downloads](https://stedolan.github.io/jq/download/)
   - Azure CLI version 2.37.0

   > **Note**: If needed, reinstall the Git for Windows and, during installation, ensure that the Git Credential Manager is enabled.

   > **Note**: If needed, upgrade the Azure CLI version by launching Command Prompt as administrator and running `az upgrade`.

   > **Note**: Following the installation of Git, ensure to set the global configuration variables user.email and user.name by running the following commands from the Git Bash shell (replace the `<your-email-address>` and `<your-full-name>` placeholders with your email address and your full name):

   ```bash
   git config --global user.email "<your-email-address>"
   git config --global user.name "<your-full-name>"
   ```

   > **Note**: To install Apache Maven, extract the content of the .zip file by running `unzip apache-maven-3.8.5-bin.zip`. Next, add the path to the bin directory of the extracted content to the PATH environment variable. Assuming that you extracted the content directly into your home directory, you could accomplish this by running the following command from the Git Bash shell:

   ```bash
   export PATH=~/apache-maven-3.8.5/bin:$PATH
   ```

   > **Note**: To install JDK, follow the instructions provided in [JDK Installation Guide](https://docs.oracle.com/en/java/javase/18/install/installation-jdk-microsoft-windows-platforms.html). Following the installation, ensure to set the JAVA_HOME environment variable to the location of the installation binaries by running the following command from the Git Bash shell:

   ```bash
   export JAVA_HOME="/c/Program Files/Java/jdk-18.0.1.1"
   ```

   > **Note**: To set up jq, download the executable to the /bin subfolder (you might need to create it) of the current user's profile folder and rename the executable to jq.exe.

### Create an Azure Spring Apps service

As the next step, you will create an Azure Spring Apps Service instance. You will use for this purpose Azure CLI. If you are interested in accomplishing this programmatically, review the Microsoft documentation that describes the provisioning process.


- [Guidance on Azure Spring Apps creation](https://docs.microsoft.com/azure/spring-cloud/quickstart-provision-service-instance?tabs=Azure-CLI&pivots=programming-language-java)

<details>
<summary>hint</summary>
<br/>

1. On your lab computer, start open the Git Bash window and, from the Git Bash prompt, run the following command to sign in to your Azure subscription:

   ```bash
   az login
   ```

1. Executing the command will automatically open a web browser window prompting you to authenticate. Once prompted, sign in using the user account that has the Owner role in the target Azure subscription that you will use in this lab and close the web browser window.

1. From the Git Bash prompt, run the following command to add the Azure Spring Apps Azure CLI extension:

   ```bash
   az extension add --name spring
   ```

1. Run the following command to update this extension to its latest version:

   ```bash
   az extension update --name spring
   ```

   > **Note**: If you receive the message `No updates available for 'spring-cloud', simply proceed to the next step.

1. Run the following commands to create a resource group that will contain all of your resources (replace the `<azure_region>` placeholder with the name of any Azure region in which you can create a Standard SKU instance of the Azure Spring Apps service and an Azure Database for MySQL Single Server instance):

   ```bash
   RANDOM=$(openssl rand -hex 3)
   RESOURCE_GROUP=springcloudlab_rg_$RANDOM
   LOCATION=<azure_region>
   az group create -g $RESOURCE_GROUP -l $LOCATION
   ```

1. Run the following commands to create an instance of the standard SKU of the Azure Spring Apps service. Note that the name of the service needs to be globally unique, so adjust it accordingly in case the randomly generated name is already in use. Keep in mind that the name can contain only lowercase letters, numbers and hyphens.

   ```bash
   SPRING_CLOUD_SERVICE=springcloudsvc$RANDOM$RANDOM
   az spring create --name $SPRING_CLOUD_SERVICE \
                          --resource-group $RESOURCE_GROUP \
                          --location $LOCATION \
                          --sku Standard
   ```

   > **Note**: This will automatically register the Microsoft.AppPlatform provider if needed.

   > **Note**: Wait for the provisioning to complete. This might take about 5 minutes.

1. Run the following command to set your default resource group name and Spring Apps service name. By setting these defaults, you don't need to repeat these names in the subsequent commands.

   ```bash
   az config set defaults.group=$RESOURCE_GROUP defaults.spring-cloud=$SPRING_CLOUD_SERVICE
   ```

1. Open a web browser window and navigate to the Azure portal. If prompted, sign in using the user account that has the Owner role in the target Azure subscription that you will use in this lab.

1. In the Azure portal, use the **Search resources, services, and docs** text box to search for and navigate to the resource group you just created.

1. On the resource group overview pane, verify that the resource group contains an Azure Spring Apps instance.

   > **Note**: In case you don't see the Azure Spring Apps service in the overview list of the resource group, select the **Refresh** toolbar button to refresh the view of the resource group resources.

1. Select the Azure Spring Apps instance and, in the vertical navigation menu, in the **Settings** section, select **Apps**. Note that the instance does not include any spring apps at this point. You will perform the app deployment later in this exercise.

</details>

### Set up the config server

Azure Spring Apps service provides a config server for the use of Spring apps. As part of its setup, you need to link it to git repo. The current configuration used by the Spring microservices resides in [the PetClinic GitHub repo](https://github.com/spring-petclinic/spring-petclinic-microservices/blob/master/spring-petclinic-config-server/src/main/resources/application.yml). You will need to create your own private git repo in this exercise, since, in one of its steps, you will be changing some of the configuration settings. 

<details>
<summary>hint</summary>
<br/>

1. On your lab computer, start a web browser and navigate to [GitHub](https://github.com) and sign in to your GitHub account. If you do not have a GitHub account, create one by navigating to [the Join GitHub page](https://github.com/join) and following the instructions provided on [the Signing up for a new GitHub account page](https://docs.github.com/en/get-started/signing-up-for-github/signing-up-for-a-new-github-account).

1. In your GitHub account, navigate to the **Repositories** page and create a new private repository named **spring-petclinic-microservices**. 

   > **Note**: Make sure to configure the repository as private.

1. On the newly created repository page, review the section titled **... or push an existing repository from the command line**.
    
    > **Note**: Record the value of the URL of the newly created GitHub repository. The value should be in the format `https://github.com/<your-github-username>/spring-petclinic-microservices-private.git`, where the `<your-github-username>` placeholder represents your GitHub user name).
    
1. On your lab computer, in the Git Bash window, run the following commands to clone the [Spring Petclinic](https://github.com/spring-petclinic/spring-petclinic-microservices) application to your workstation:

   ```bash
   rm spring-petclinic-microservices/ -fr
   git clone https://github.com/spring-petclinic/spring-petclinic-microservices.git
   ```

1. From the Git Bash prompt, run the following commands to change the working directory to the one containing the cloned repository and then push its content to your private GitHub repository (where the `<your-github-username>` placeholder represents your GitHub user name):

   ```bash
   cd ~/spring-petclinic-microservices/
   git remote remove origin
   git remote add origin https://github.com/<your-github-username>/spring-petclinic-microservices-private.git
   git branch -M main
   git push -u origin main
   ```

1. When prompted to sign in to GitHub, select the **Sign in with your browser** option. This will automatically open a new tab in the web browser window, prompting you to provide your GitHub username and password.

1. In the browser window, enter your GitHub credentials, select **Sign in**, and, once successfully signed in, close the newly opened browser tab.

1. From the Git Bash prompt, run the following commands to copy all the config server configuration yaml files from [spring-petclinic-microservices-config](https://github.com/spring-petclinic/spring-petclinic-microservices-config) to the local folder on your lab computer.

   ```bash
   curl -o admin-server.yml https://raw.githubusercontent.com/spring-petclinic/spring-petclinic-microservices-config/main/admin-server.yml
   curl -o api-gateway.yml https://raw.githubusercontent.com/spring-petclinic/spring-petclinic-microservices-config/main/api-gateway.yml
   curl -o application.yml https://raw.githubusercontent.com/spring-petclinic/spring-petclinic-microservices-config/main/application.yml
   curl -o customer-service.yml https://raw.githubusercontent.com/spring-petclinic/spring-petclinic-microservices-config/main/customer-service.yml
   curl -o discovery-server.yml https://raw.githubusercontent.com/spring-petclinic/spring-petclinic-microservices-config/main/discovery-server.yml
   curl -o tracing-server.yml https://raw.githubusercontent.com/spring-petclinic/spring-petclinic-microservices-config/main/tracing-server.yml
   curl -o vets-service.yml https://raw.githubusercontent.com/spring-petclinic/spring-petclinic-microservices-config/main/vets-service.yml
   curl -o visit-service.yml https://raw.githubusercontent.com/spring-petclinic/spring-petclinic-microservices-config/main/visit-service.yml
   ```

1. From the Git Bash prompt, run the following commands to commit and push your changes to your private GitHub repository.

   ```bash
   git add .
   git commit -m 'added base config'
   git push
   ```

</details>

Once you completed the initial update of your git repository hosting the server configuration, you need to set up the config server for your Azure Spring Apps instance. As part of the setup process, you need to create a Personal Access Token (PAT) in your GitHub repo and make it available to the config server.

[Guidance on config server setup](https://docs.microsoft.com/azure/spring-cloud/quickstart-setup-config-server?tabs=Azure-CLI&pivots=programming-language-java)
[Guidance for a private repo with basic authentication](https://docs.microsoft.com/azure/spring-cloud/how-to-config-server#private-repository-with-basic-authentication)
[Guidance for creating a PAT](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

<details>
<summary>hint</summary>
<br/>

1. To create a PAT, switch to the web browser window displaying your private GitHub repository, select the avatar icon in the upper right corner, and then select **Settings**.

1. At the bottom of the vertical navigation menu, select **Developer settings**, select **Personal access tokens**, and then select **Generate new token**.

1. If prompted to confirm access, enter your GitHub account password and select **Confirm password**.

1. On the **New personal access token** page, in the **Note** text box, enter a descriptive name, such as **spring-petclinic-config-server-token**.

1. Ensure that value in the **Expiration** drop-down list is set to **30 days**.

1. In the **Select scopes** section, select **repo** and then select **Generate token**.

1. Record the generated token. You will need it in the next step.

1. Switch to the Git Bash prompt and run the following commands to set the environment variables hosting your GitHub repository and GitHub credentials (replace the `<git_repository>`, `<git_username>`, and `<git_password>` placeholders with the URL of your GitHub repository, the name of your GitHub user account, and the newly generated PAT value, respectively).

   > **Note**: The URL of the GitHub repository should be in the format `https://github.com/<your-github-username>/spring-petclinic-microservices-private.git`, where the `<your-github-username>` placeholder represents your GitHub user name).

   ```bash
   GIT_REPO=<git_repository>
   GIT_USERNAME=<git_username>
   GIT_PASSWORD=<git_password>
   ```

1. To set up the config server such that it points to your GitHub repository, from the Git Bash prompt, run the following command. 

   ```bash
   az spring-cloud config-server git set --name $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --uri $GIT_REPO \
                           --label main \
                           --password $GIT_PASSWORD \
                           --username $GIT_USERNAME 
   ```

   > **Note**: Wait for the operation to complete. This might take about 2 minutes.

</details>

### Create an Azure MySQL Database service

You now have the compute service that will host your applications and the config server that will be used by your migrated application. Before you start deploying individual microservices as Azure Spring Apps applications, you need to first create an Azure Database for MySQL Single Server-hosted database for them. To accomplish this, you can use the following guidance.

[Create MySQL Single Server and Database](https://docs.microsoft.com/azure/mysql/quickstart-create-mysql-server-database-using-azure-cli)

You will also need to update the config for your applications to use the newly provisioned MySQL Server to authorize access to your private GitHub repository. This will involve updating the application.yml config file in your private git config repo with the values provided in the MySQL Server connection string.

<details>
<summary>hint</summary>
<br/>

1. Run the following commands to create an instance of Azure Database for MySQL Single Server. Note that the name of the server must be globally unique, so adjust it accordingly in case the randomly generated name is already in use. Keep in mind that the name can contain only lowercase letters, numbers and hyphens. In addition, replace the `<myadmin_password>` placeholder with a complex password and record its value. 

   ```bash
   SQL_SERVER_NAME=springcloudmysql$RANDOM$RANDOM
   SQL_ADMIN_PASSWORD=<myadmin_password>
   DATABASE_NAME=petclinic

   az mysql server create \
         --admin-user myadmin \
         --admin-password ${SQL_ADMIN_PASSWORD} \
         --name ${SQL_SERVER_NAME} \
         --resource-group ${RESOURCE_GROUP}  \
         --sku-name GP_Gen5_2  \
         --version 5.7 \
         --storage-size 5120
   ```

   > **Note**: Wait for the provisioning to complete. This might take about 3 minutes.

1. Once the Azure Database for MySQL Single Server instance gets created, it will output details about its settings. In the output, you will find the server connection string. Record its value since you will need it later in this exercise. 

1. Run the following commands to create a database in the Azure Database for MySQL Single Server instance.

   ```bash
   az mysql db create --server-name $SQL_SERVER_NAME \
         --resource-group $RESOURCE_GROUP \
         --name $DATABASE_NAME
   ```

1. You will also need to allow connections to the server from Azure Spring Apps. For now, to accomplish this, you will create a server firewall rule to allow inbound traffic from all Azure Services. This way your apps running in Azure Spring Apps will be able to reach the MySQL database providing them with persistent storage. In one of the upcoming exercises, you will restrict this connectivity to limit it exclusively to the apps hosted by your Azure Spring Apps instance. 

   ```bash
   az mysql server firewall-rule create --name allAzureIPs \
       --server ${SQL_SERVER_NAME} \
       --resource-group ${RESOURCE_GROUP} \
       --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
   ```

1. From the Git Bash window, in the config repository you cloned locally, use your favorite text editor to open the application.yml file. Change the entries in lines 82, 83, and 84 that contain the values of the target datasource endpoint, the corresponding admin user account, and its password. Set these values by using the information in the Azure Database for MySQL Single Server connection string you recorded earlier in this task. Your configuration should look like this:

   > **Note**: The original content of these three lines in the application.yml file has the following format:

   ```yaml
       url: jdbc:mysql://localhost:3306/db?useSSL=false
       username: root
       password: petclinic
   ```

   > **Note**: The updated content of these three lines in the application.yml file should have the following format (where the `<mysql-server-name>` and `<myadmin-password>` placeholders represent the name of the Azure Database for MySQL Single Server instance and the password you assigned to the myadmin account during its provisioning, respectively):

   ```yaml
       url: jdbc:mysql://<mysql-server-name>.mysql.database.azure.com:3306/db?useSSL=true
       username: myadmin@<mysql-server-name>
       password: <myadmin-password>
   ```

   > **Note**: Ensure to change the value of the `useSSL` parameter to `true`, since this is enforced by default by Azure Database for MySQL Single Server.

1. Save the changes and push the updates you made to the **application.yml** file to your private GitHub repo by running the following commands from the Git Bash prompt:

   ```bash
   git add .
   git commit -m 'azure mysql info'
   git push
   ```

</details>

   > **Note**: At this point, the admin account user name and password are stored in clear text in the application.yml config file. In one of upcoming exercises, you will remediate this potential vulnerability by removing clear text credentials from your configuration.

### Deploy the Spring Petclinic app components to the Spring Apps service

You now have the compute and data services available for deployment of the components of your applications, including spring-petclinic-admin-server, spring-petclinic-customers-service, spring-petclinic-vets-service, spring-petclinic-visits-service and spring-petclinic-api-gateway. In this task, you will deploy these components as microservices to the Azure Spring Apps service. You will not be deploying the spring-petclinic-config-server and spring-petclinic-discovery-server to Azure Spring Apps, since these will be provided to you by the platform. To perform the deployment, you can use the following guidance:

[Guidance on creating apps on Azure Spring Apps](https://docs.microsoft.com/azure/spring-cloud/quickstart-deploy-apps?tabs=Azure-CLI&pivots=programming-language-java)

   > **Note**: The spring-petclinic-api-gateway and spring-petclinic-admin-server will have a public endpoint assigned to them.

   > **Note**: When you deploy the customers-service, vets-service and visits-service you should do so with the mysql profile activated.

<details>
<summary>hint</summary>
<br/>

1. You will start by building all the microservice of the spring petclinic application. To accomplish this, run `mvn clean package` in the root directory of the application.

   ```bash
   cd ~/spring-petclinic-microservices/
   mvn clean package -DskipTests
   ```

1. Verify that the build succeeds by reviewing the output of the `mvn clean package -DskipTests` command, which should have the following format: 

   ```bash
   [INFO] Reactor Summary for spring-petclinic-microservices 2.6.3:
   [INFO]
   [INFO] spring-petclinic-microservices ..................... SUCCESS [  0.224 s]
   [INFO] spring-petclinic-admin-server ...................... SUCCESS [  5.665 s]
   [INFO] spring-petclinic-customers-service ................. SUCCESS [  4.231 s]
   [INFO] spring-petclinic-vets-service ...................... SUCCESS [  3.152 s]
   [INFO] spring-petclinic-visits-service .................... SUCCESS [  2.902 s]
   [INFO] spring-petclinic-config-server ..................... SUCCESS [  1.030 s]
   [INFO] spring-petclinic-discovery-server .................. SUCCESS [  1.429 s]
   [INFO] spring-petclinic-api-gateway ....................... SUCCESS [  8.277 s]
   [INFO] ------------------------------------------------------------------------
   [INFO] BUILD SUCCESS
   [INFO] ------------------------------------------------------------------------
   [INFO] Total time:  27.310 s
   [INFO] Finished at: 2022-05-12T18:43:06Z
   [INFO] ------------------------------------------------------------------------
   ```

1. For each application you will now create an app on Azure Spring Apps service. You will start with the api-gateway. To deploy it, from the Git Bash prompt, run the following command:

   ```bash
   az spring-cloud app create --service $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --name api-gateway \
                           --assign-endpoint true
   ```

   > **Note**: Wait for the provisioning to complete. This might take about 5 minutes.

1. Next deploy the jar file to this newly created app by running the following command from the Git Bash prompt:

   ```bash
   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --name api-gateway \
                           --no-wait \
                           --artifact-path spring-petclinic-api-gateway/target/spring-petclinic-api-gateway-2.6.1.jar
   ```

1. In the same way create an app for the admin-server microservice:

   ```bash
   az spring-cloud app create --service $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --name app-admin \
                           --assign-endpoint
   ```

   > **Note**: Wait for the operation to complete. This might take about 5 minutes.

1. Next deploy the jar file to this newly created app:

   ```bash
   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --name app-admin \
                           --no-wait \
                           --artifact-path spring-petclinic-admin-server/target/spring-petclinic-admin-server-2.6.1.jar
   ```

1. Next, you will create an app for the customers-service microservice:

   ```bash
   az spring-cloud app create --service $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --name customers-service 
   ```

   > **Note**: Wait for the operation to complete. This might take about 5 minutes.

1. For the customers service you will not assign an endpoint but you will set the mysql profile:

   ```bash
   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --name customers-service \
                           --no-wait \
                           --artifact-path spring-petclinic-customers-service/target/spring-petclinic-customers-service-2.6.1.jar \
                           --env SPRING_PROFILES_ACTIVE=mysql
   ```

1. Next, you will create an app for the visits-service microservice:


   ```bash
   az spring-cloud app create --service $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --name visits-service 
   ```

   > **Note**: Wait for the operation to complete. This might take about 5 minutes.

1. For the visit-service will also skip the endpoint assignment but include the mysql profile:

   ```bash
   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --name visits-service \
                           --no-wait \
                           --artifact-path spring-petclinic-visits-service/target/spring-petclinic-visits-service-2.6.1.jar \
                           --env SPRING_PROFILES_ACTIVE=mysql
   ```

1. To conclude, you will create an app for the vets-service microservice:


   ```bash
   az spring-cloud app create --service $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --name vets-service 
   ```

   > **Note**: Wait for the operation to complete. This might take about 5 minutes.

1. In this case you will also skip the endpoint assignment but include the mysql profile:

  ```bash
   az spring-cloud app deploy --service $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --name vets-service \
                           --no-wait \
                           --artifact-path spring-petclinic-vets-service/target/spring-petclinic-vets-service-2.6.1.jar \
                           --env SPRING_PROFILES_ACTIVE=mysql
   ```

</details>

### Test the application through the publicly available endpoint

Now that you have deployed all of your microservices, verify that the application is accessible via a web browser.

<details>
<summary>hint</summary>
<br/>

1. To list all deployed apps, from the Git Bash shell, run the following CLI statement, which will also list all publicly accessible endpoints:

   ```bash
   az spring-cloud app list --service $SPRING_CLOUD_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --output table
   ```

1. Alternatively, you can switch to the web browser window displaying the Azure portal interface, navigate to your Azure Spring Apps instance and select **Apps** from the vertical navigation menu. In the list of apps, select **api-gateway**, on the **api-gateway \| Overview** page, note the value of the **URL** property.

1. Open another web browser tab and navigate to the URL of the api-gateway endpoint to display the application web interface. 

</details>

#### Review

In this exercise, you migrated your existing Spring Petclinic microservices application to Azure Spring Apps.
