---
lab:
    title: 'Lab: Configure Azure Event Hubs for Azure Spring Apps'
    module: 'Module 6: Configure Azure Event Hubs for Azure Spring Apps'
---

# Challenge: Configure Azure Event Hubs for Azure Spring Apps
# Student manual

## Challenge scenario

You have completed implement messaging functionality for the Spring Petclinic application. Now, you will implement the event processing functionality by integrating your application with Event Hub.

## Objectives

After you complete this challenge, you will be able to:

- Create an Azure Event Hub resource
- Use an existing microservice to send events to the Event Hub resource
- Update an existing microservice to receive Event Hub events
- Inspect telemetry data being received

## Challenge Duration

- **Estimated Time**: 60 minutes

## Instructions

During this challenge, you'll:

- Create an Azure Event Hub resource
- Use an existing microservice to send events to the Event Hub resource
- Update an existing microservice to receive Event Hub events
- Inspect telemetry data being received

### Create Event Hub resource

You will first need to create an Azure Event Hub namespace to send events to. Create an Event Hub namespace and assign to it a globally unique name. In the namespace you will then create an eventhub named **telemetry**. You can use the following guidance to implement these changes.

[Quickstart: Create an event hub using Azure CLI](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-quickstart-cli).

You should add the connection string to the telemetry event hub in your Key Vault so the microservices can safely retrieve this value.

   > **Note**: As an alternative you can use the Managed Identity of your microservice to connect to the eventhub. For this challenge however you will store the connection string in your Key Vault. You can use the following guidance to implement these changes.

[Authenticate a managed identity with Azure Active Directory to access Event Hubs Resources](https://docs.microsoft.com/en-us/azure/event-hubs/authenticate-managed-identity?tabs=latest).

The connection to the eventhub needs to be stored in the **spring.kafka.properties.sasl.jaas.config** application property. Store its value in a Key Vault secret named **SPRING-KAFKA-PROPERTIES-SASL-JAAS-CONFIG**.

<details>
<summary>hint</summary>
<br/>

1. On your lab computer, in Git Bash window, from the Git Bash prompt, run the following command to create an Event Hub namespace. The name you use for your namespace should be globally unique, so adjust it accordingly in case the randomly generated name is already in use. 

   ```bash
   EVENTHUBS_NAMESPACE=javalab-eh-ns-$RANDOM$RANDOM

   az eventhubs namespace create \
     --resource-group $RESOURCE_GROUP \
     --name $EVENTHUBS_NAMESPACE \
     --location $LOCATION
   ```

1. Next, create an eventhub named **telemetry** in the newly created namespace.

   ```bash
   EVENTHUB_NAME=telemetry

   az eventhubs eventhub create \
     --name $EVENTHUB_NAME \
     --resource-group $RESOURCE_GROUP \
     --namespace-name $EVENTHUBS_NAMESPACE
   ```

1. Create a new authorization rule for sending and listening to the telemetry eventhub.

   ```bash
   RULE_NAME=listensendrule

   az eventhubs eventhub authorization-rule create \
     --resource-group $RESOURCE_GROUP \
     --namespace-name $EVENTHUBS_NAMESPACE \
     --eventhub-name $EVENTHUB_NAME \
     --name $RULE_NAME \
     --rights Listen Send
   ```

1. Retrieve the connection string for this authorization rule in an environment variable.

   ```bash
   EVENTHUB_CONNECTIONSTRING=$(az eventhubs eventhub authorization-rule keys list \
       --resource-group $RESOURCE_GROUP \
       --namespace-name $EVENTHUBS_NAMESPACE \
       --eventhub-name $EVENTHUB_NAME \
       --name $RULE_NAME \
       --query primaryConnectionString \
       --output tsv)
   ```

1. Display the value of the connection string and verify that it only allows access to your telemetry eventhub.

   ```bash
   echo $EVENTHUB_CONNECTIONSTRING
   ```

   > **Note**: The connection string should have the following format (where the `<event-hub-namespace>` placeholder represents the name of your Event Hub namespace and the `<shared-access-key>` placeholder represents a Shared Access Signature value corresponding to the listensendrule access key):

   ```txt
   Endpoint=sb://<event-hub-namespace>.servicebus.windows.net/;SharedAccessKeyName=listensendrule;SharedAccessKey=<shared-access-key>;EntityPath=telemetry
   ```

1. From the Git Bash window, in your local application repository, use your favorite text editor to create a file named **secretfile.txt** with the following content and replace the `<connection-string>` placeholder with the value of the connection string you displayed in the previous step, excluding the trailing string `;EntityPath=telemetry`:

   ```txt
   org.apache.kafka.common.security.plain.PlainLoginModule required username="$ConnectionString" password="<connection-string>";
   ```

1. Save the file

1. Create a new Key Vault secret for this connection string.

   ```bash
   az keyvault secret set \
       --name SPRING-KAFKA-PROPERTIES-SASL-JAAS-CONFIG \
       --file secretfile.txt \
       --vault-name $KEYVAULT_NAME
   ```

1. In your configuration repository's **application.yml** file, add the kafka configuration in the **spring** section by appending the following YAML fragment (make sure to replace the `<eventhub-namespace>` placeholder in the value of the **bootstrap-servers** parameter):

   ```yaml
     kafka:
       bootstrap-servers: <eventhub-namespace>.servicebus.windows.net:9093
       client-id: first-service
       group-id: $Default
       properties:
         sasl.jaas.config: 
         sasl.mechanism: PLAIN
         security.protocol: SASL_SSL
         spring.json:
           use.type.headers: false
           value.default.type: com.targa.labs.dev.telemetrystation.Message
   ```

   > **Note**: The resulting content should have the following format:

   ```yaml
   spring:
     config:
       activate:
         on-profile: mysql
     jms:
       servicebus:
         connection-string: ${spring.jms.servicebus.connectionstring}
         idle-timeout: 60000
         pricing-tier: premium
     datasource:
       schema: classpath*:db/mysql/schema.sql
       data: classpath*:db/mysql/data.sql
       url: jdbc:mysql://javaopenlabmysql.mysql.database.azure.com:3306/db?useSSL=true
       initialization-mode: ALWAYS
     kafka:
       bootstrap-servers: <eventhub-namespace>.servicebus.windows.net:9093
       client-id: first-service
       group-id: $Default
       group.id: $Default
       properties:
         sasl.mechanism: PLAIN
         security.protocol: SASL_SSL
         spring.json:
           use.type.headers: false
           value.default.type: com.targa.labs.dev.telemetrystation.Message
   topic:
     name: telemetry
   azure:
     keyvault:
       enabled: true
       uri: https://springcloudlab2-kv.vault.azure.net/
   ```

1. Commit and push your changes to the remote repository.

   ```bash
   git add .
   git commit -m 'added event hub'
   git push
   ```

</details>

### Use an existing microservice to send events to the Event Hub resource

You will now implement the functionality that will allow you to emulate sending events from a third party system to the telemetry Event Hub. You can find this third party system in the **extra** folder of the starter project. 

   > **Note**: This folder contains the content of the **producer** directory available from https://github.com/Azure/azure-event-hubs-for-kafka/tree/master/quickstart/java**

Edit the **producer.config** file in the **extra/src/main/resources** folder: 
- change the **bootstrap.servers** config setting so it contains the name of the Event Hub namespace you provisioned earlier in this lab
- change the **sasl.jaas.config** config setting so it contains the connection string to the **telemetry** event hub

Update the **TestProducer.java** file in the **extra/src/main/java** directory, so it uses **telemetry** as a topic name.

Compile the producer app. You will use it at the end of this lab to send 100 events to your event hub. 

<details>
<summary>hint</summary>
<br/>

1. From the Git Bash window, in your local application repository, use your favorite text editor to open the **spring-petclinic-microservices/extra/src/main/resources/producer.config** file. Change line 1 by replacing the `mynamespacename` placeholder with the name of the Event Hub namespace you provisioned earlier in this lab.

   ```yaml
   bootstrap.servers=mynamespace.servicebus.windows.net:9093
   ```

1. Change line 4 by replacing the placeholder representing the Event Hub connection string placeholder with the value of the connection string to the **telemetry** event hub. This value should match the output of the **$EVENTHUB_CONNECTIONSTRING** environment variable.

   ```yaml
   sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="$ConnectionString"    password="Endpoint=sb://mynamespace.servicebus.windows.net/;SharedAccessKeyName=XXXXXX;SharedAccessKey=XXXXXX";

   Endpoint=sb://javalab-eh-ns-3080227949.servicebus.windows.net/;SharedAccessKeyName=listensendrule;SharedAccessKey=OOqTHuZYbLoQ4CLlNAD9e4R6viBOUx9QWLasXsFqnZI=;EntityPath=telemetry
   ```

1. Save the change to the file.

1. Open the **TestProducer.java** file in the **spring-petclinic-microservices/extra/src/main/java** directory. Alter line 16 by replacing the `test` placeholder with **telemetry** so it uses the telemetry event hub and save the change.

   ```java
       private final static String TOPIC = "telemetry";
   ```

1. From the Git Bash window, set the current working directory to the **extra** folder and run a maven build.

   ```bash
   mvn clean package
   ```

</details>

### Update an existing microservice to receive Event Hub events

In this task, you will update the customers microservice to receive events from the telemetry event hub. You can use the following guidance to implement these changes.

[Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/html/)

<details>
<summary>hint</summary>
<br/>

1. In the Git Bash window, in your local application repository, use your favorite text editor to open the **pom.xml** file of the **spring-petclinic-customers-service** microservice, add to it another dependency element within the `<!-- Spring Cloud -->` section of the `<dependencies> element, and save the change:

   ```xml
           <dependency>
               <groupId>org.springframework.kafka</groupId>
               <artifactId>spring-kafka</artifactId>
           </dependency>
   ```

1. In the **spring-petclinic-microservices/spring-petclinic-customers-service/src/main/java/org/springframework/samples/petclinic/customers** folder, create a directory named **services**. Next, in this directory, create a **EventHubListener.java** class file with the following code:

   ```java
   package org.springframework.samples.petclinic.customers.services;

   import org.slf4j.Logger;
   import org.slf4j.LoggerFactory;
   import org.springframework.kafka.annotation.KafkaListener;
   import org.springframework.stereotype.Service;

   @Service
   public class EventHubListener {

      private static final Logger log = LoggerFactory.getLogger(EventHubListener.class);

      @KafkaListener(topics = "telemetry", groupId = "$Default")
        public void receive(String in) {
           log.info("Received message from kafka queue: {}",in);
           System.out.println(in);
       }
   } 
   ```

   > **Note**: This class uses the **KafkaListener** annotation to start listening to an event hub using the **$Default** group of the **telemetry** event hub. The received messages are written to the log as info messages.

1. In the Git Bash window, navigate back to the root folder of the spring petclinic repository and rebuild the application.

   ```bash
   cd ~/spring-petclinic-microservices/
   mvn clean package -DskipTests
   ```

1. Redeploy the customers-service microservice to Azure Spring Cloud.

   ```bash
   az spring-cloud app deploy \
     --service $SPRING_CLOUD_SERVICE \
     --resource-group $RESOURCE_GROUP \
     --name customers-service \
     --no-wait \
     --artifact-path spring-petclinic-customers-service/target/spring-petclinic-customers-service-2.6.1.jar \
     --env SPRING_PROFILES_ACTIVE=mysql
   ```

</details>

### Inspect telemetry data being received

To conclude this lab, you will run the producer app to send 100 events to your event hub and use output logs of the customers microservice to verify that these messages are being received. You can use the following guidance to implement these changes.

[Log streaming](https://docs.microsoft.com/en-us/azure/spring-cloud/quickstart-logs-metrics-tracing?tabs=Azure-CLI&pivots=programming-language-java#log-streaming-1)

<details>
<summary>hint</summary>
<br/>

1. In the Git Bash window, set the current working directory to the **extra** folder and run the TestProducer application.

   ```bash
   mvn exec:java -Dexec.mainClass="TestProducer"
   ```

1. Verify that the output indicates that 100 events were sent to the **telemetry** event hub.

1. Press the Ctrl+C key combination to return to the command prompt.

1. From the same Git Bash window, run the following command to start the log stream output for the **customers-service**.

   ```bash
   az spring-cloud app logs -f --service $SPRING_CLOUD_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --name customers-service
   ```

1. Review the output and verify that it contains the output that has the following format:

   ```txt
   2022-05-27 12:26:10.520  INFO 1 --- [ntainer#0-0-C-1] o.s.s.p.c.services.EventHubListener      : Received message from kafka queue: Test Data #1 
   Test Data #1
   ```

1. Switch to the web browser displaying the Azure portal, navigate to the page of the resource group containing resources you provisioned in this lab, and select the entry representing your Event Hub namespace. 

1. On the Event Hub namespace page, in the navigation menu, in the **Entities** section, select **Event Hubs** and then select the **telemetry** event hub entry.

1. On the **Overview** page, review the **Messages** graph to verify that it includes metrics representing incoming and outgoing messages. 

</details>

#### Review

In this lab, you implemented support for event processing by Azure Spring Apps applications.
