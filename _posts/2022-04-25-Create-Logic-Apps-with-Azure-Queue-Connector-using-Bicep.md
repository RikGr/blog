---
layout: post
title:  Use Bicep to create a Logic App with an Azure Queue connector
author: Rik Groenewoud
---


## How it started 
On April 1st I started my new job at Xpirit as an Azure & DevOps Consultant. As part of the Managed Services team I will focus on implementing and monitoring Azure infrastructure. 
During my second week my teammates said: "Create the Infrastructure as Code (IaC) for the Logic Apps". "It is going to be fun", they said...
And yes, fun it was, especially when I finally made it work! 

It all started pretty easy, some of the bicep fot the Logic App was already there so I did not have to start from scratch. Although I never used Logic Apps before and certainly did not create them by code, I was pretty confident I could make this work.

Soon enough I had the Azure resources available and the Logic Apps that did not use the Azure Storage Queue was working like a charm. However, the Logic Apps with the *Azure Queues connector* kept on failing. In the Logic App Designer this was the error I saw: 

![Error](/images/blog-1.1.png)

This was the bicep I had for the Logic App in question: 

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


## How it was going
 
So how did I troubleshoot: 

<ul>
  <li><strong>Documentation</strong> In the Microsoft documentation I read that with Access Key authentication it should be possible to make this work. So the theory was there. Unfortunately I could not find any code examples or references to this particular Azure Queues connector. I kept on changing the bicep code here and there and tried build after build to no avail. I started to wonder if it was even possible to do this using bicep. 
  </li>
  <li><strong>Github and StackOverflow</strong> I consulted the community and although finding some good help and informational blogs (see References below) I still could not make it work.
  </li>
  <li><strong>Asking my colleagues</strong> I posted the question on our internal Azure Slack channel and sure enough after an initial silence some of my colleagues came to the rescue. They suggested to build the solution via the portal, export it as .JSON, decompile it into bicep and see if I could make that work. 
  Although I already did some testing and comparing the JSON, I decided to give it a try once more including the bicep decompile using <code>az bicep decompile</code>.
  And yes, now I came somewhere. In my own test environment I managed to build and deploy the Logic App with a *working* Azure Queue connection. 
  </li>
</ul>

## How it was fixed

So now I knew it could work. What remained was an exercise in comparing the two templates. And yes, after some initial trial and error it dawned on me that it must be something in the definition of the API Connection in the Logic App. So I changed some of the *working* bicep to the way I was doing it with a variable for the id: 

```bicep
connection: {
                name: azureQueueConnectionId
              }
```

And there it was! The same error was thrown.

So I finally concluded that the parameter **connection name** must be defined in the instance code itself and as a *string* to make it all work:

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

With this change I managed to completely automate the deployment of the Logic App as it was designed. Mission Accomplished.

## References

 <a href="https://docs.microsoft.com/en-us/connectors/azurequeues/"> Microsoft Docs on Logic App Azure Queue Connector</a> 

 <a href="https://checinski.cloud/azure-logic-app-blob-storage-connection-bicep/"> During troubleshooting I found this nice blog on using Bicep and Logic Apps</a>
