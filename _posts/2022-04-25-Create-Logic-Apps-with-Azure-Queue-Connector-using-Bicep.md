---
layout: post
title:  Use Bicep to create Logic Apps with an Azure Queue connector
author: Rik Groenewoud
tags: microsoft azure bicep iac logicapp azurequeue
---


## How it started 
On April 1st, I started my new job at [Xpirit](https://www.xpirit.com) as an Azure & DevOps Consultant. As part of the Managed Services team I will focus on implementing and monitoring Azure infrastructure. 
During my second week my teammates said: "Create the Infrastructure as Code (IaC) for the Logic Apps". It will be fun they said...
And yes, fun it was, especially when I finally made it work! 

It all started pretty easy, some of the Bicep fot the Logic App was already in place so I did not have to start from scratch. The templates and reference for the Logic App and API Connection can be found respectively [here](https://docs.microsoft.com/en-us/azure/templates/microsoft.logic/workflows?tabs=bicep) and [here](https://docs.microsoft.com/en-us/azure/templates/microsoft.web/connections?tabs=bicep). But in this current scenario where we had to *migrate existing* infrastructure to Bicep I always suggest to export the existing resource via the portal to have a blueprint of how the code should look like. This can be done via the Azure Portal under the Automation options:

![Portal](/images/blog-1.2.png)

Soon enough I had the Azure resources available and the Logic App *without the Azure Storage Queue Connector* were working just fine. However, the Logic Apps with the Azure Queues connector in their design kept on failing. In the Logic App Designer in the Azure Portal this was the error I saw: 

![Error](/images/blog-1.1.png)

A snippet from the Bicep I had for the Logic App in question: 

```bicep
actions: {
        'Put_a_message_on_a_queue_(V2)' : {
          runafter: {}
          type: 'ApiConnection'
          inputs: {
            body: 'start'
            host: {
              connection: {
                name: azureQueueConnectionId
              }
            }
              method: 'post'
              path: '/v2/storageAccounts/${storageAccountName}/queues/dailymaintenance/messages'
            
          }
```

Notice that for **connection name** I used a variable <code>azureQueueConnectionId</code>. This variable refers to a parameter <code>azureQueueConnectionIdParameter</code> and this parameter was declared in the main deployment bicep file as follows: 
```
azureQueueConnectionIdParameter: logicAppConnection.outputs.id 
```
This way of referring to external resources and using output from the Logic App is normal practice within Bicep so I thought this was the best way to do it.

## Troubleshoot time
So how did I troubleshoot?

<ul>
  <li><strong>Documentation</strong> In the Microsoft documentation I read that with Access Key authentication it should be possible to make this work. So the theory was there. Unfortunately I could not find any code examples or references to this particular Azure Queues connector. I kept on changing the code here and there and tried build after build to no avail. I started to wonder if it was even possible to do this using Bicep. 
  </li>
  <li><strong>Github and StackOverflow</strong> I consulted the community through [Bicep Github](https://github.com/Azure/bicep) and asked around on StackOverflow. Although finding some good help and informational blogs (see References below) I still could not make it work.
  </li>
  <li><strong>Asking my colleagues</strong> I posted the question on our internal Azure Slack channel and sure enough after an initial silence some of my colleagues came to the rescue. They suggested to build the solution via the portal, export it as .JSON, decompile it into Bicep and see if I could make that work. 
  Although I already did some testing and comparing the JSON, I decided to give it a try once more including the decompile option using <code>az bicep decompile</code>.
  And yes, now I came somewhere. In my own test environment I managed to build and deploy the Logic App with a *working* Azure Queue connection. 
  </li>
</ul>

## How it was fixed
Now I knew it could work. What remained was an exercise in comparing the two templates. And yes, after some initial trial and error it dawned on me that it must be something in the definition of the API Connection in the Logic App. So I changed some of the *working* Bicep to the variable for the connection name (which is actually the API Connection id): 

```bicep
connection: {
                name: azureQueueConnectionId
              }
```

And there it was! Now the same error was thrown.

So after this test, I knew I was looking in the right direction. I decided to replace the variable I was using with the syntax from the working example.  
The parameter **$connections**  now is declared in the resource itself and the **connention name** refers to this param as a *string*:

```
      actions: {
        'Put_a_message_on_a_queue_(V2)' : {
          runafter: {}
          type: 'ApiConnection'
          inputs: {
            body: 'start'
            host: {
              connection: {
                name: '@parameters(\'$connections\')[\'azurequeues\'][\'connectionId\']'
              }
            }
              method: 'post'
              path: '/v2/storageAccounts/${storageAccountName}/queues/dailymaintenance/messages'
            
          }
        }
      }
    }
    parameters: {
      '$connections': {
        value: {
          azurequeues: {
            connectionId: logicAppConnection.id
            connectionName: 'LogicAppConnection'
            id: '/subscriptions/xxxxxxxxxxx/providers/Microsoft.Web/locations/westeurope/managedApis/azurequeues'
          }
        }
      }
    }

```
Although to me it is not entirely clear why, everything worked as expected with the above syntax. For now I am happy with the result. In the future I would like to investigate further to see if I can replace this parameter towards the main deployment file to keep the bicep clean and readable. 

## Other References
- [Microsoft Docs on Logic App Azure Queue Connector](https://docs.microsoft.com/en-us/connectors/azurequeues/)
- [During troubleshooting I found this nice blog on using Bicep and Logic Apps](https://checinski.cloud/azure-logic-app-blob-storage-connection-bicep/>)

