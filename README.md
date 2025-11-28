# Consume Azure Service Bus messages in Power Platform

This project demonstrates how a Power Automate flow can be used to consume messages from an Azure Service Bus queue, reliably process the messages and make changes in Dataverse. This pattern can be used to integrate external system with Dataverse.

## Service bus

For experimentation, a basic tier Service Bus instance will do. It's still not completely free but there's no hourly cost. It's advised to turn off the main flow if you are not actively working on it to keep the overall number of Service Bus operations to a minimum. This proof-of-concept expects a queue named `contacts`. For authentication, create a Shared access policy with the Send and Listen claims. In the Azure portal select your service bus resource, settings -> shared access policies -> click add. You will use the primary connection string displayed inside the SAS policy to connect your Power Automate flow to the Service Bus queue.

## Power platform solutions

The main solution `servicebusanddataverse` consists of a few components only:

* two connection references, one for Dataverse and one for Service Bus.
* three flows, two for manually publishing messages and the main one is to consume and process messages.

### Export and store solution in source control

Useful pac commands:

    pac solution list
    pac solution export --name servicebusanddataverse --overwrite
    pac solution unpack --zipfile servicebusanddataverse.zip --folder ./Solutions/servicebusanddataverse

### Recreate unmanged solution and import into your own environment

    pac solution pack --zipfile servicebusanddataverse.zip --folder ./Solutions/servicebusanddataverse

The resulting .zip can be imported into your environment. It has an implicit dependency on the standard contact table having a column called `ltornyi_entityid` which also has an alternate key defined. For convenience, this is captured in the `Externalids` solution. Using `pac`, recreate and import this solution before attempting to import the `servicebusanddataverse` solution.

During the import of the solution, you will be asked to choose/set up connections for the service bus and for Dataverse. For the service bus connection, the simplest is to use the SAS policy you created previously but other options are also available. For the Dataverse connection, using your own Power Platform credentials is the simplest but you can also use a service principal connection, see setup instructions [here](https://learn.microsoft.com/en-us/power-automate/dataverse/manage-dataverse-connections#use-service-principal-connections-with-the-dataverse-connector).

### Description of the main flow

The flow consumes messages from the contact queue and creates or updates the relevant contact record in Dataverse. Details:

The trigger is set to `When one or more messages arrive in a queue (peek-lock)`; queue name is `contacts` and Maximum message count is 50. Important: `Split on` was set to `off` and Concurrency control is also `Off`. This combination results in one flow run for each bucket of 50 messages (or less) on the queue. This is not exactly what is happening; see more details below under Example test runs.

The main block of the flow is an `apply to each` action on the trigger output. For each message received, the loop

* decodes the incoming message using `base64ToString`
* parses the decoded message as JSON that expects 3 string properties: lastName, firstName and entityID
* finds the first contact in the contact table by filtering on entityid
* if the contact is found, it updates the first and last name columns
* if the contact is not found, it creates the contact using the 3 properties received in the message
* complete the service bus message i.e. removes it from the queue

The above actions are in a Scope called Try. We also have another Scope called Catch inside the `apply to each` action which runs after the Try Scope if Try has timed out or has failed. This is a well-established try-catch error handling pattern used in Power Automate flows.

In the Catch Scope, we have a few actions that take care of error handling, specifically

* extracts output of the Try Scope and captures the action that failed or timed out
* stores the name and error output of the action in an array
* moves the current service bus message into the dead-letter queue

After the Apply to each action

* a condition checks if we have at least one error in the array
* if yes, we execute a compose action so that the error array content becomes part of the run log
* execute a Terminate flow action that sets the flow run's status to Failed

### Example test runs

With the flow trigger settings described above (*Maximum message count* = 50, *Split on* set to `Off` and *Concurrency control* set to `Off`), depending on the number of messages on the queue, we observed the following flow runs:

| number of messages waiting in the queue | concurrent flow runs | flow runs processing messages |
| ----------- | ----------- | ----------- |
| 10 | 2 | 1, 2-10 |
| 50 | 2 | 1, 2-49 |
| 140 | 4 | 1, 2-51, 52-101, 102-140 |

Consistently in each case, there was one flow run that only processed 1 message but the rest of the runs picked up the next 50 messages (or less) from the queue.

When Concurrency was turned on, more flow runs were generated and typically a message was processed more than once in parallel flow run instances. This is obviously not optimal; probably more investigation is needed.