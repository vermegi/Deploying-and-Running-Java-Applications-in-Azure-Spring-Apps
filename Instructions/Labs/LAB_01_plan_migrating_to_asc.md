---
lab:
    title: 'Lab: Plan for migrating an existing microservices application to Azure'
    module: 'Module 1: Plan for migrating an existing microservices application to Azure'
---

# Lab: Plan for migrating an existing microservices application to Azure
# Student lab manual

## Lab scenario

You want to establish a plan for migrating your existing Spring Petclinic microservices application to Azure. 

## Objectives

After you complete this lab, you will be able to:

- Examine the application components based on the information provided in its GitHub repository
- Identify the Azure services most suitable for hosting your application
- Identify the Azure services most suitable for storing data of your application
- Identify resource organization in Azure
- Identify tools for connecting to and managing your Azure environment

## Lab Duration

- **Estimated Time**: 30 minutes

## Instructions

During this lab, you'll:
- Examine the application components based on the information provided in its GitHub repository
- Consider the Azure services most suitable for hosting your application
- Consider the Azure services most suitable for storing data of your application
- Consider resource organization in Azure
- Consider tools for connecting to and managing your Azure environment

This first lab will be mainly a conceptual exercise that does not involve deploying any of the application components to Azure. You will run the initial deployment in the next exercise.

### Examine the application components based on the information provided in its GitHub repository

To start, you will learn about the existing Spring Petclinic application.

1. Start a web browser, navigate to the [GitHub repo hosting the Spring Petclinic application code](https://github.com/spring-petclinic/spring-petclinic-microservices) and review its [README.md file](https://github.com/spring-petclinic/spring-petclinic-microservices/blob/master/README.md#understanding-the-spring-petclinic-application).

1. Examine the information about [starting services locally without Docker](https://github.com/spring-petclinic/spring-petclinic-microservices/blob/master/README.md#starting-services-locally-without-docker), [Starting services locally with docker-compose](https://github.com/spring-petclinic/spring-petclinic-microservices/blob/master/README.md#starting-services-locally-with-docker-compose), and [Starting services locally with docker-compose and Java](https://github.com/spring-petclinic/spring-petclinic-microservices/blob/master/README.md#starting-services-locally-with-docker-compose-and-java). If time permits, consider launching the application locally using either of these methods.

1. In the web browser displaying the GitHub repo, navigate to each folder containing the code of the individual spring-petclinic-* services and review their content. You don't need to know their full details, but you should understand their basic structure.

### Consider the Azure services most suitable for hosting your application

Now that you have familiarized yourself with the application you will be migrating to Azure, as the next step, you will need to consider different compute options you have at your disposal for hosting this application.

The three primary options you will take into account are Azure App Service, Azure Kubernetes Service and Azure Spring Cloud. Given that the Spring Petclinic application consists of multiple microservices working together to provide the functionality you reviewed in the previous task, what would you consider to be the most suitable option? Before you answer this question, review the following requirements:

* The Spring Petclinic application should be accessible via a public endpoint to any user (anonymously).
* The new implementation of Spring Petclinic should eliminate the need to manually upgrade and manage the underlying infrastructure. Instead, the application should use the platform-as-a-service (PaaS) model.
* Spring Petclinic implementation needs to adhere to the principles of the microservices architecture, with each component of the application running as a microservice and granular control over cross-component communication. The application will evolve into a solution that will provide automatic and independent scaling of each component and extend to include additional microservices.

Consider any additional steps you may need to perform to migrate the Spring Petclinic application to the target service.

Fill out the following table based on your analysis:

||Azure App Service|Azure Kubernetes Service|Azure Spring Cloud|
|---|---|---|---|
|Public endpoint available||||
|Auto-upgrade underlying hardware||||
|Run microservices||||
|Additional advantages||||
|Additional disadvantages||||

<details>
<summary>hint</summary>
<br/>

* Each of the 3 options supports a public endpoint that can be access anonymously.
* Each of the 3 options supports automatic upgrades and eliminates the need to manage the underlying infrastructure.
  * With Azure App Service, upgrades are automatic. All underlying infrastructure is managed by the platform.
  * With Azure Kubernetes Service (AKS), you can enable automatic upgrades based on the channel of your choice (patch, stable, rapid, node-image). The underlying infrastructure consists of VM's that you provision as part of agent pools, however you don't manage them directly. 
  * With Azure Spring Cloud, all tasks related to upgrading and managing the underlying infrastructure are taken care of by the platform. While Azure Spring Cloud is built on top of an AKS cluster, that cluster is fully managed.
* Both AKS and Azure Spring Cloud offer a convenient approach to implementing the microservices architecture. They also provide support for Spring Boot applications. If you decided to choose Azure App Service, you would need to create a new web app instance for each microservice, while both AKS and Azure Cloud Spring require only a single instance. AKS also facilitates controlling traffic flow between microservices by using network policies.
* Azure Spring Cloud Service offers an easy migration path for existing spring boot applications. This would be an advantage for your existing application.
* Azure Spring Cloud Service eliminates any administrative overhead required to run a Kubernetes cluster. This simplifies the operational model.
* AKS would require an extra migration step that involves containerizing all components. You will also need to implement Azure Container Registry to store and deploy your container images from.
* Running and operating an AKS cluster introduces an additional effort.
* Azure App Service scalability is more limited than AKS or Azure Spring Cloud Service. 

Given the above constraints and feature sets, in the case of the Spring Petclinic application, Azure Spring Cloud and Azure Kubernetes Service represent the most viable implementation choices. 

</details>

### Consider the Azure services most suitable for storing data of your application

Now that you identified the viable compute platforms, you need to decide which Azure service could be used to store the applications data.

Azure platform offers several database-as-a-services options, including Azure SQL Database, Azure Database for MySQL, Azure Cosmos DB, and Azure Database for PostgreSQL. Your choice of the database technology should be based on the following requirements for the Spring Petclinic application:

* The target database service should simplify the migration path from the on-premises MySQL deployment. 
* The target database service must support automatic backups.
* The target database service needs to support automatic patching.

Based on these requirements, you decided to use Azure Database for MySQL Single Server.

### Consider resource organization in Azure

You now have a clear understanding of which Azure services you will have working together for the first stage of migrating of the Spring Petclinic application. Next, you need to plan how the resource will be organized in Azure (without actually creating these resources yet, since that will be part of the next exercise). To address this need, try to answer the following questions:

- How many resource groups will you be creating for hosting your Azure resources?

<details>
<summary>hint</summary>
<br/>
In Azure all resources that are created and deleted together typically should belong to the same resource group. In this case, since there is 1 application which provides a specific functionality, you can provision all resources for this application in a single resource group.
</details>

- How will you configure networking for the application components?

<details>
<summary>hint</summary>
<br/>
In case you chose to use Azure Spring Cloud, you have the option to deploy Azure Spring Cloud either into a virtual network or deploy it without a virtual network dependency. The latter approach will simplify the task of making the first migrated version of the application accessible from the internet. Later on, in one of the subsequent exercises, you will change this approach to accommodate additional requirements. For now though, for the sake of simplicity, you will not create any virtual networks for Azure Spring Cloud.

In case you chose AKS as the hosting platform, you will need at least one subnet in a virtual network to run the nodes of your AKS cluster. This subnet for now can be small, such as /26, which allows for the total of 64 IP addresses (although some of them are pre-allocated for the platform use).

The Azure Database for MySQL deployment will not require any virtual network connectivity for the first phase of the migration of the application. This will also change in one of the subsequent exercises, when you will implement additional security measures to protect the full application stack.
</details>

- Are there any supporting services you would need for running the application?

<details>
<summary>hint</summary>
<br/>
In case you chose Azure Spring Cloud, no additional supporting services are needed during the first phase of the migration. All you need is a compute platform and a database.

In case you chose AKS, you will also need a container registry for storing any container images that will be deployed to the cluster. You can use for this purpose Azure Container Registry.
</details>

### Consider tools for connecting to and managing your Azure environment

You have now identified the resources you will need to proceed with the first stage of the migration and determined the optimal way of organizing them. Next, you need to consider how you will connect to your Azure environment. Ask yourself the following questions:

- What tools would you need for connecting to the Azure platform?

<details>
<summary>hint</summary>
<br/>
For connecting to the Azure platform, you can use either the [Azure portal](http://portal.azure.com), or command line tools such as [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/what-is-azure-cli). The latter might be more challenging, but it will facilitate scripting your setup and making it repeatable in case anything needs to change or recreated.
In your lab environment, make sure you can log into the Azure portal by using the credentials that were provided to you for running the lab.

It is also a good idea to double check whether Azure CLI was correctly installed in your lab environment by running the following command from the Git Bash shell window:

```bash
az --help
```

There are other tools you will us as well (including Git and mvn), but the portal and Azure CLI will be the primary ones you will be using during the initial deployment of your application into Azure.
</details>

You also should record any commands and scripts you execute for later reference. This will help you in the subsequent exercises, in case you need to reuse them to repeat the same sequence of steps.

- What additional tools would you need to perform the migration?

<details>
<summary>hint</summary>
<br/>
In case you chose Azure Spring Cloud as the target platform, there are no additional tools needed for your to perform the migration steps.

In case you chose AKS as the target platform, you will also need Docker tools to containerize the microservices that the application consists of. You will also need to consider the most optimal base image for containerizing the microservices. 
</details>

With all of the above questions answered, you now have a good understanding of the steps and resources needed to perform your migration. In the next exercise you will execute its first phase.

#### Review

In this lab, you established a plan for migrating your existing Spring Petclinic microservices application to Azure. 