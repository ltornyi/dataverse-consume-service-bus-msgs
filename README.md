# Consume Azure Service Bus messages in Power Platform

This project demonstrates how a Power Automate flow can be used to consume messages from an Azure Service Bus queue, reliably process the messages and make changes in Dataverse. This pattern can be used to integrate external system with Dataverse.

## Service bus

For experimentation, a basic tier Service Bus instance will do. It's still not completely free but there's no hourly cost. It's advised to turn off the main flow if you are not actively working on it to keep the overall number of Service Bus operations to a minimum. This proof-of-concept expects a queue named `contacts`. For authentication, create a Shared access policy with the Send and Listen claims. In the Azure portal select your service bus resource, settings -> shared access policies -> click add. You will use the primary connection string displayed inside the SAS policy to connect your Power Automate flow to the Service Bus queue.

## Power platform solution

The solution consists of a few components only:

* two connection references, one for Dataverse and one for Service Nus.
* three flows, two for manually publishing messages and the main one is to consume and process messages.

