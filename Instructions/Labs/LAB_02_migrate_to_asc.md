---
lab:
    Title: 'Challenge 02: Migrate a Spring Apps application to Azure'
    Learn module: 'Learn Module 2: Migrate a Spring Apps application to Azure'
---

# Challenge 02: Migrate a Spring Apps application to Azure

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

The below image illustrates the end state you will be building in this challenge.

![Challenge 2 architecture](./images/asa-openlab-2.png)

## Challenge Duration

- **Estimated Time**: 120 minutes

## Instructions

During this challenge, you will:

- Create an Azure Spring Apps service
- Set up the config server
- Create an Azure MySQL Database service
- Deploy the Spring Petclinic app components to the Spring Apps service
- Provide a publicly available endpoint for the Spring Petclinic application
- Test the application through the publicly available endpoint

> **Note**: The labstarter branch of the Azure-Samples/spring-petclinic-microservices repository contains a dev container for Java development. This container contains all the needed tools for running this lab. In case you want to use this dev container you can either use a [GitHub CodeSpace](https://github.com/features/codespaces) in case your GitHub account is enabled for Codespaces. Or you can use the [Visual Studio Code Remote Containers option](https://code.visualstudio.com/docs/remote/containers).

> **Note**: In case you want to run this lab on your own workstation, this lab contains guidance for a Windows workstation. Your workstation should contain the following components:

- Visual Studio Code available from [Visual Studio Code Downloads](https://code.visualstudio.com/download)
  - Java and Spring Boot Visual Studio Code extension packs available from [Java extensions for Visual Studio Code](https://code.visualstudio.com/docs/java/extensions)
- Git for Windows 2.3.61 available from [Git Downloads](https://git-scm.com/downloads), or similar on another OS.
  - **Note**: If needed, reinstall Git and, during installation, ensure that the Git Credential Manager is enabled.
- [Apache Maven 3.8.5](apache-maven-3.8.5-bin.zip) available from [Apache Maven Project downloads](https://maven.apache.org/download.cgi)
  - **Note**: To install Apache Maven, extract the content of the .zip file by running `unzip apache-maven-3.8.5-bin.zip`. Next, add the path to the bin directory of the extracted content to the `PATH` environment variable. Assuming that you extracted the content directly into your home directory, you could accomplish this by running the following command from the Git Bash shell: `export PATH=~/apache-maven-3.8.5/bin:$PATH`.
- Java 8 and the Java Development Kit (JDK) available from [JDK downloads](https://aka.ms/download-jdk/microsoft-jdk-17.0.5-windows-x64.msi)
  - **Note**: To install JDK on Windows, follow the instructions provided in [JDK Installation Guide](https://learn.microsoft.com/en-us/java/openjdk/install#install-on-windows). Make sure to use the `FeatureJavaHome` feature during the install to update the `JAVA_HOME` environment variable.
- In case you prefer to use IntelliJ IDEA as an IDE instead of Visual Studio Code: Azure Toolkit for IntelliJ IDEA 3.51.0 from the IntelliJ Plugins UI from [IntelliJ IDEA](https://www.jetbrains.com/idea/download/#section=windows)
- Azure CLI version 2.37.0
  - **Note**: If needed, upgrade the Azure CLI version by launching Command Prompt as administrator and running `az upgrade`.
- jq command line tool available from [JQ Downloads](https://stedolan.github.io/jq/)
  - **Note**: To set up jq, download the executable to the /bin subfolder (you might need to create it) of the current user's profile folder and rename the executable to jq.exe if running on Windows.

> **Note**: Following the installation of Git, ensure to set the global configuration variables `user.email` and `user.name` by running the following commands from the Git Bash shell (replace the `<your-email-address>` and `<your-full-name>` placeholders with your email address and your full name):

```bash
git config --global user.email "<your-email-address>"
git config --global user.name "<your-username>"
```

### Create an Azure Spring Apps service

As the next step, you will create an Azure Spring Apps Service instance. You will use for this purpose Azure CLI. If you are interested in accomplishing this programmatically, review the Microsoft documentation that describes the provisioning process.

- [Guidance on Azure Spring Apps creation](https://docs.microsoft.com/azure/spring-cloud/quickstart-provision-service-instance?tabs=Azure-CLI&pivots=programming-language-java)

<details>
<summary>hint</summary>
<br/>

1. On your lab computer, open the Git Bash window and, from the Git Bash prompt, run the following command to sign in to your Azure subscription:

   ```bash
   az login
   ```

1. Executing the command will automatically open a web browser window prompting you to authenticate. Once prompted, sign in using the user account that has the Owner role in the target Azure subscription that you will use in this lab and close the web browser window.

1. Make sure that you are logged in to the right subscription for the consecutive commands.

   ```bash
   az account list -o table
   ```

1. If in the above statement you don't see the right account being indicated as your default one, change your environment to the right subscription with the following command, replacing the `<subscription-id>`.

   ```bash
   az account set --subscription <subscription-id>
   ```

1. Run the following commands to create a resource group that will contain all of your resources (replace the `<azure-region>` placeholder with the name of any Azure region in which you can create a Standard SKU instance of the Azure Spring Apps service and an Azure Database for MySQL Single Server instance, see [this page](https://azure.microsoft.com/global-infrastructure/services/?products=mysql,spring-apps&regions=all) for regional availability details of those services):

   ```bash
   UNIQUEID=$(openssl rand -hex 3)
   APPNAME=petclinic
   RESOURCE_GROUP=rg-$APPNAME-$UNIQUEID
   LOCATION=<azure-region>
   az group create -g $RESOURCE_GROUP -l $LOCATION
   ```
1. Run the following command to add the spring extension.

   ```bash
   az extension add --name spring
   ``` 
    
1. Run the following commands to create an instance of the standard SKU of the Azure Spring Apps service. Note that the name of the service needs to be globally unique, so adjust it accordingly in case the randomly generated name is already in use. Keep in mind that the name can contain only lowercase letters, numbers and hyphens.

   ```bash
   SPRING_APPS_SERVICE=sa-$APPNAME-$UNIQUEID
   az spring create --name $SPRING_APPS_SERVICE \
                    --resource-group $RESOURCE_GROUP \
                    --location $LOCATION \
                    --sku Standard
   ```

   > **Note**: This will also create for you an Application Insights resource. This Application Insights resource is created still in `classic` mode and not in the newer `workspace` mode. If the region you are deploying to doesn't support this `classic` mode anymore, the CLI will show a warning to say it skipped App Insights creation and you should assign it manually. Don't worry in case you see this message though, it will not influence the rest of the lab for you. We will cover monitoring in depth in a next module.

   > **Note**: Wait for the provisioning to complete. This might take about 5 minutes.

1. Run the following command to set your default resource group name and Spring Apps service name. By setting these defaults, you don't need to repeat these names in the subsequent commands.

   ```bash
   az config set defaults.group=$RESOURCE_GROUP defaults.spring=$SPRING_APPS_SERVICE
   ```

1. Open a web browser window and navigate to the Azure portal. If prompted, sign in using the user account that has the Owner role in the target Azure subscription that you will use in this lab.

1. In the Azure portal, use the **Search resources, services, and docs** text box to search for and navigate to the resource group you just created.

1. On the resource group overview pane, verify that the resource group contains an Azure Spring Apps instance.

   > **Note**: In case you don't see the Azure Spring Apps service in the overview list of the resource group, select the **Refresh** toolbar button to refresh the view of the resource groups.

   > **Note**: You will notice an Application Insights resource also was created in your resource group. You will use this in one of the next labs.

1. Select the Azure Spring Apps instance and, in the vertical navigation menu, in the **Settings** section, select **Apps**. Note that the instance does not include any spring apps at this point. You will perform the app deployment later in this exercise.

</details>

### Set up the config server


Azure Spring Apps service provides a config server for the use of Spring apps. As part of its setup, you need to link it to a git repo. The current configuration used by the Spring microservices resides in the [labstarter branch of the spring-petclinic-microservices-config repo](https://github.com/Azure-Samples/spring-petclinic-microservices-config/tree/labstarter). You will need to create your own private git repo in this exercise, since, in one of its steps, you will be changing some of the configuration settings.

As part of the setup process, you need to create a Personal Access Token (PAT) in your GitHub repo and make it available to the config server. It is important that you make note of the PAT after it has been created.

- [Guidance for creating a PAT](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

<details>
<summary>hint</summary>
<br/>

1. On your lab computer, in your web browser, navigate to your GitHub account, navigate to the **Repositories** page and create a new private repository named **spring-petclinic-microservices-config**.

   > **Note**: Make sure to configure the repository as private.

1. To create a PAT, select the avatar icon in the upper right corner, and then select **Settings**.

1. At the bottom of the vertical navigation menu, select **Developer settings**, select **Personal access tokens**, and then select **Generate new token**.

1. On the **New personal access token** page, in the **Note** text box, enter a descriptive name, such as **spring-petclinic-config-server-token**.

   > **Note**: There is a new **Beta** experience available on GitHub for more fine-grained access tokens. This experience will create a token with a more limited scope than full repository scope (which basically gives access to all your repositories). The lab will work as well with a more fine-grained token, in that case, in the **Fine-grained tokens (Beta)** token creation page, choose for **Only select repositories** and select your config repository. For the **Repository permissions** select for the **Contents** the **Read-only** access level. You can use this fine-grained token when you configure your config-server on Azure Spring Apps. We recommend you create a second token in case you also need a personal access token for interacting with the repositories from the Git Bash prompt.

1. In the **Select scopes** section, select **repo** and then select **Generate token**.

1. Record the generated token. You will need it in this and subsequent labs.

1. From the Git Bash prompt, change the current directory to the **projects** folder. Next, clone the newly created GitHub repository by typing `git clone `, pasting the clone URL you copied into Clipboard in the previous step, and entering the PAT string followed by the `@` symbol in front of `github.com`.

   ```bash
   cd ~/projects
   # Clone config repo
   git clone https://<token>@github.com/<your-github-username>/spring-petclinic-microservices-config.git
    
   # Clone source code repo
   git clone https://<token>@github.com/<your-github-username>/spring-petclinic-microservices.git

   ```

    > **Note**: Make sure to replace the `<token>` and `<your-github-username>` placeholders in the URL listed above with the value of the GitHub PAT and your GitHub user name when running the `git clone` command.

1. From the Git Bash prompt, change the current directory to the newly created **spring-petclinic-microservices-config** folder and run the following commands to copy all the config server configuration yaml files from the [labstarter branch of the spring-petclinic-microservices-config repo](https://github.com/Azure-Samples/spring-petclinic-microservices-config/tree/labstarter) to the local folder on your lab computer.

   ```bash
   cd spring-petclinic-microservices-config
   curl -o admin-server.yml https://raw.githubusercontent.com/Azure-Samples/spring-petclinic-microservices-config/labstarter/admin-server.yml
   curl -o api-gateway.yml https://raw.githubusercontent.com/Azure-Samples/spring-petclinic-microservices-config/labstarter/api-gateway.yml
   curl -o application.yml https://raw.githubusercontent.com/Azure-Samples/spring-petclinic-microservices-config/labstarter/application.yml
   curl -o customers-service.yml https://raw.githubusercontent.com/Azure-Samples/spring-petclinic-microservices-config/labstarter/customers-service.yml
   curl -o discovery-server.yml https://raw.githubusercontent.com/Azure-Samples/spring-petclinic-microservices-config/labstarter/discovery-server.yml
   curl -o tracing-server.yml https://raw.githubusercontent.com/Azure-Samples/spring-petclinic-microservices-config/labstarter/tracing-server.yml
   curl -o vets-service.yml https://raw.githubusercontent.com/Azure-Samples/spring-petclinic-microservices-config/labstarter/vets-service.yml
   curl -o visits-service.yml https://raw.githubusercontent.com/Azure-Samples/spring-petclinic-microservices-config/labstarter/visits-service.yml
   ```

1. From the Git Bash prompt, run the following commands to commit and push your changes to your private GitHub repository.

   ```bash
   git add .
   git commit -m 'added base config'
   git push
   ```

1. In your web browser, refresh the page of the newly created **spring-petclinic-microservices-config** repository and double check that all the configuration files are there.

</details>

### Set up the config server for Azure Spring Apps
    
Once you completed the initial update of your git repository hosting the server configuration, you need to set up the config server for your Azure Spring Apps instance. 

- [Guidance on config server setup](https://docs.microsoft.com/azure/spring-cloud/quickstart-setup-config-server?tabs=Azure-CLI&pivots=programming-language-java).
- [Guidance for a private repo with basic authentication](https://docs.microsoft.com/azure/spring-cloud/how-to-config-server#private-repository-with-basic-authentication).

<details>
<summary>hint</summary>
<br/>

1. Switch to the Git Bash prompt and run the following commands to set the environment variables hosting your GitHub repository and GitHub credentials (replace the `<git-repository>`, `<git-username>`, and `<git-PAT>` placeholders with the URL of your GitHub repository, the name of your GitHub user account, and the newly generated PAT value, respectively).

   > **Note**: The URL of the GitHub repository should be in the format `https://github.com/<your-github-username>/spring-petclinic-microservices-config.git`, where the `<your-github-username>` placeholder represents your GitHub user name.

   ```bash
   GIT_REPO=<git-repository>
   GIT_USERNAME=<git-username>
   GIT_PASSWORD=<git-PAT>
   ```

1. To set up the config server such that it points to your GitHub repository, from the Git Bash prompt, run the following command.

   ```bash
   az spring config-server git set \
                           --name $SPRING_APPS_SERVICE \
                           --resource-group $RESOURCE_GROUP \
                           --uri $GIT_REPO \
                           --label main \
                           --password $GIT_PASSWORD \
                           --username $GIT_USERNAME 
   ```

   > **Note**: In case you are using a branch other than `main` in your config repo, you can change the branch name with the `label` parameter.

   > **Note**: Wait for the operation to complete. This might take about 2 minutes.

</details>

### Create an Azure MySQL Database service

You now have the compute service that will host your applications and the config server that will be used by your migrated application. Before you start deploying individual microservices as Azure Spring Apps applications, you need to first create an Azure Database for MySQL Single Server-hosted database for them. To accomplish this, you can use the following guidance:

- [Quickstart: Create an Azure Database for MySQL Flexible Server using Azure CLI](https://learn.microsoft.com/azure/mysql/flexible-server/quickstart-create-server-cli).

You will also need to update the config for your applications to use the newly provisioned MySQL Server to authorize access to your private GitHub repository. This will involve updating the application.yml config file in your private git config repo with the values provided in the MySQL Server connection string.

<details>
<summary>hint</summary>
<br/>

1. Run the following commands to create an instance of MySQL Flexible server. Note that the name of the server must be globally unique, so adjust it accordingly in case the randomly generated name is already in use. Keep in mind that the name can contain only lowercase letters, numbers and hyphens. In addition, replace the `<myadmin-password>` placeholder with a complex password and record its value.

   ```bash
   MYSQL_SERVER_NAME=mysql-$APPNAME-$UNIQUEID
   MYSQL_ADMIN_USERNAME=myadmin
   MYSQL_ADMIN_PASSWORD=<myadmin-password>
   DATABASE_NAME=petclinic
   
    az mysql flexible-server create \
        --admin-user myadmin \
        --admin-password ${MYSQL_ADMIN_PASSWORD} \
        --name ${MYSQL_SERVER_NAME} \
        --resource-group ${RESOURCE_GROUP} 
   ```

   > **Note**: During the creation you will be asked whether access for your IP address should be added and whether access for all IP's should be added. Answer `n` for no on both questions.

   > **Note**: Wait for the provisioning to complete. This might take about 3 minutes.

1. Once the Azure Database for MySQL Single Server instance gets created, it will output details about its settings. In the output, you will find the server connection string. Record its value since you will need it later in this exercise.

1. Run the following commands to create a database in the Azure Database for MySQL Single Server instance.

   ```bash
    az mysql flexible-server db create \
        --server-name $MYSQL_SERVER_NAME \
        --resource-group $RESOURCE_GROUP \
        -d $DATABASE_NAME
   ```

1. You will also need to allow connections to the server from Azure Spring Apps. For now, to accomplish this, you will create a server firewall rule to allow inbound traffic from all Azure Services. This way your apps running in Azure Spring Apps will be able to reach the MySQL database providing them with persistent storage. In one of the upcoming exercises, you will restrict this connectivity to limit it exclusively to the apps hosted by your Azure Spring Apps instance.

   ```bash
    az mysql flexible-server firewall-rule create \
        --rule-name allAzureIPs \
        --name ${MYSQL_SERVER_NAME} \
        --resource-group ${RESOURCE_GROUP} \
        --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
   ```

1. From the Git Bash window, in the config repository you cloned locally, use your favorite text editor to open the application.yml file. Change the entries in lines 82, 83, and 84 that contain the values of the target datasource endpoint, the corresponding admin user account, and its password. Set these values by using the information in the Azure Database for MySQL Single Server connection string you recorded earlier in this task. Your configuration should look like this:

   > **Note**: The original content of these three lines in the application.yml file have the following format:

   ```yaml
       url: jdbc:mysql://localhost:3306/db?useSSL=false
       username: root
       password: petclinic
   ```

   > **Note**: The updated content of these three lines in the **application.yml** file should have the following format (where the `<mysql-server-name>`, `<myadmin-password>` and `<mysql-database-name>` placeholders represent the name of the Azure Database for MySQL Single Server instance, the password you assigned to the myadmin account during its provisioning, and the name of the database i.e. `petclinic`, respectively):

   ```yaml
       url: jdbc:mysql://<mysql-server-name>.mysql.database.azure.com:3306/<mysql-database-name>?useSSL=true
       username: myadmin
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

You now have the compute and data services available for deployment of the components of your applications, including `spring-petclinic-admin-server`, `spring-petclinic-customers-service`, `spring-petclinic-vets-service`, `spring-petclinic-visits-service` and `spring-petclinic-api-gateway`. In this task, you will deploy these components as microservices to the Azure Spring Apps service. You will not be deploying the `spring-petclinic-config-server` and `spring-petclinic-discovery-server` to Azure Spring Apps, since these will be provided to you by the platform. To perform the deployment, you can use the following guidance:

- [Guidance on creating apps on Azure Spring Apps](https://docs.microsoft.com/azure/spring-cloud/quickstart-deploy-apps?tabs=Azure-CLI&pivots=programming-language-java).

   > **Note**: The `spring-petclinic-api-gateway` and `spring-petclinic-admin-server` will have a public endpoint assigned to them.

   > **Note**: When you deploy the `customers-service`, `vets-service` and `visits-service` you should do so with the `mysql` profile activated.

<details>
<summary>hint</summary>
<br/>

1. In the parent **pom.xml** file double check the version number on line 9.

    ```bash
        <parent>        
            <groupId>org.springframework.samples</groupId>
            <artifactId>spring-petclinic-microservices</artifactId>
            <version>2.7.6</version>    
        </parent>
    ```

1. From the Git Bash window, set a `VERSION` environment variable to this version number `2.7.6`.

   ```bash
   VERSION=2.7.6
   ```

1. You will start by building all the microservice of the spring petclinic application. To accomplish this, run `mvn clean package` in the root directory of the application.

   ```bash
   cd ~/projects/spring-petclinic-microservices
   mvn clean package -DskipTests
   ```

1. Verify that the build succeeds by reviewing the output of the `mvn clean package -DskipTests` command, which should have the following format:

   ```bash
   [INFO] ------------------------------------------------------------------------
   [INFO] Reactor Summary for spring-petclinic-microservices 2.7.6:
   [INFO] 
   [INFO] spring-petclinic-microservices ..................... SUCCESS [  0.274 s]
   [INFO] spring-petclinic-admin-server ...................... SUCCESS [  6.462 s]
   [INFO] spring-petclinic-customers-service ................. SUCCESS [  4.486 s]
   [INFO] spring-petclinic-vets-service ...................... SUCCESS [  1.943 s]
   [INFO] spring-petclinic-visits-service .................... SUCCESS [  2.026 s]
   [INFO] spring-petclinic-config-server ..................... SUCCESS [  0.885 s]
   [INFO] spring-petclinic-discovery-server .................. SUCCESS [  0.960 s]
   [INFO] spring-petclinic-api-gateway ....................... SUCCESS [  6.022 s]
   [INFO] ------------------------------------------------------------------------
   [INFO] BUILD SUCCESS
   [INFO] ------------------------------------------------------------------------
   [INFO] Total time:  24.584 s
   [INFO] Finished at: 2022-11-29T13:31:17Z
   [INFO] ------------------------------------------------------------------------
   ```

1. For each application you will now create an app on Azure Spring Apps service. You will start with the `api-gateway`. To deploy it, from the Git Bash prompt, run the following command:

   ```bash
   az spring app create \
            --service $SPRING_APPS_SERVICE \
            --resource-group $RESOURCE_GROUP \
            --name api-gateway \
            --assign-endpoint true
   ```

   > **Note**: Wait for the provisioning to complete. This might take about 5 minutes.

1. Next deploy the jar file to this newly created app by running the following command from the Git Bash prompt:

   ```bash
   az spring app deploy \
            --service $SPRING_APPS_SERVICE \
            --resource-group $RESOURCE_GROUP \
            --name api-gateway \
            --no-wait \
            --artifact-path spring-petclinic-api-gateway/target/spring-petclinic-api-gateway-$VERSION.jar
   ```

1. In the same way create an app for the `admin-server` microservice:

   ```bash
   az spring app create \
            --service $SPRING_APPS_SERVICE \
            --resource-group $RESOURCE_GROUP \
            --name app-admin \
            --assign-endpoint true
   ```

   > **Note**: Wait for the operation to complete. This might take about 5 minutes.

1. Next deploy the jar file to this newly created app:

   ```bash
   az spring app deploy \
            --service $SPRING_APPS_SERVICE \
            --resource-group $RESOURCE_GROUP \
            --name app-admin \
            --no-wait \
            --artifact-path spring-petclinic-admin-server/target/spring-petclinic-admin-server-$VERSION.jar
   ```

1. Next, you will create an app for the `customers-service` microservice, without assigning an endpoint:

   ```bash
   az spring app create \
            --service $SPRING_APPS_SERVICE \
            --resource-group $RESOURCE_GROUP \
            --name customers-service
   ```

   > **Note**: Wait for the operation to complete. This might take about 5 minutes.

1. For the customers service you will set the `mysql` profile:

   ```bash
   az spring app deploy \
            --service $SPRING_APPS_SERVICE \
            --resource-group $RESOURCE_GROUP \
            --name customers-service \
            --no-wait \
            --artifact-path spring-petclinic-customers-service/target/spring-petclinic-customers-service-$VERSION.jar \
            --env SPRING_PROFILES_ACTIVE=mysql
   ```

1. Next, you will create an app for the `visits-service` microservice, also without an endpoint assigned:

   ```bash
   az spring app create \
               --service $SPRING_APPS_SERVICE \
               --resource-group $RESOURCE_GROUP \
               --name visits-service 
   ```

   > **Note**: Wait for the operation to complete. This might take about 5 minutes.

1. For the `visit-service` you will also include the `mysql` profile:

   ```bash
   az spring app deploy \
               --service $SPRING_APPS_SERVICE \
               --resource-group $RESOURCE_GROUP \
               --name visits-service \
               --no-wait \
               --artifact-path spring-petclinic-visits-service/target/spring-petclinic-visits-service-$VERSION.jar \
               --env SPRING_PROFILES_ACTIVE=mysql
   ```

1. To conclude, you will create an app for the `vets-service` microservice, again without an endpoint assigned:

   ```bash
   az spring app create \
               --service $SPRING_APPS_SERVICE \
               --resource-group $RESOURCE_GROUP \
               --name vets-service 
   ```

   > **Note**: Wait for the operation to complete. This might take about 5 minutes.

1. In this case you will also include the `mysql` profile:

  ```bash
   az spring app deploy \
               --service $SPRING_APPS_SERVICE \
               --resource-group $RESOURCE_GROUP \
               --name vets-service \
               --no-wait \
               --artifact-path spring-petclinic-vets-service/target/spring-petclinic-vets-service-$VERSION.jar \
               --env SPRING_PROFILES_ACTIVE=mysql
   ```

</details>

### Test the application through the publicly available endpoint

If any of the deployments failed, you can check the logs of a specific app using the following CLI statement (using `customers-service` as an example):

```bash
az spring app logs --service $SPRING_APPS_SERVICE \
                   --resource-group $RESOURCE_GROUP \
                   --name customers-service \
                   --follow
```

Now that you have deployed all of your microservices, verify that the application is accessible via a web browser.

<details>
<summary>hint</summary>
<br/>

1. To list all deployed apps, from the Git Bash shell, run the following CLI statement, which will also list all publicly accessible endpoints:

   ```bash
   az spring app list --service $SPRING_APPS_SERVICE \
                      --resource-group $RESOURCE_GROUP \
                      --output table
   ```

1. Alternatively, you can switch to the web browser window displaying the Azure portal interface, navigate to your Azure Spring Apps instance and select **Apps** from the vertical navigation menu. In the list of apps, select **api-gateway**, on the **api-gateway \| Overview** page, note the value of the **URL** property.

1. Open another web browser tab and navigate to the URL of the api-gateway endpoint to display the application web interface.

1. You can also navigate to the URL of the admin-server to see insight information of your microservices.

</details>

#### Review

In this exercise, you migrated your existing Spring Petclinic microservices application to Azure Spring Apps.
