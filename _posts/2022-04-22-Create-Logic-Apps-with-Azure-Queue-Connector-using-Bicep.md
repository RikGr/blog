---
layout: post
title:  "My first week at Xpirit, or how to use bicep to create a Logic App with a Azure Queue connector"
---

## How it started 
You can create the Infrastructure as Code (IaC) for the Logic Apps they said, it is going to be fun they said. 
And yes fun it was, at least when I finally fixed it. 

It all started pretty easy, some of the bicep for the API connection and one Logic App was already there. Although I never had used Logic Apps before and certainly did not create them by code, I was pretty confident I could make this work.
Soon enough I had the Azure resources available and the Logic App without the API connection was working like a charm. However, the Logic Apps with the Azure Queues connectors kept on failing. In the Logic App Designer this was the error I saw: 

![Error](/images/blog-1.1.png)

This was the bicep I had for the Logic App in question: 

{% highlight json %}

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
{% endhighlight %}


## How it was going
 
So how did I troubleshoot: 

<ul>
  <li>**Documentation** In the Microsoft documentation I read that with Access Key authentication it should be possible to make this work. So the theory was there. Unfortunately I could not find any code examples or references to this particular connector. I kept on changing the bicep code here and there and tried build after build to no avail. I started to wonder if this was even possible to do this using bicep. 
  </li>
  <li>**Github and StackOverflow** I consulted the community and although finding some good help and informational blogs such as:c [link] I still could not make it work
  </li>
  <li>**Asking my colleagues** I posted the question on our internal Azure Slack channel and sure enough after an initial silence some of my colleagues came to the rescue. They suggested to build the solution via the portal, export it as .JSON, decompile it into bicep and see if I could make that work. 
  Although I already did some testing and comparing the JSON, I decided to once more give it a try including the bicep decompile using <code>az bicep decompile</code>.
  And yes, now I came somewhere. In my own test environment I managed to build and deploy the Logic App with a *working* Azure Queue connection. 
  </li>
</ul>

## How it was fixed

So now I knew it could work. Now it was an exercise in comparing the two templates. And yes, after some initial trial and error it dawned on me that it must be something in the definition of the API in the Logic App. So I changed some of the *working* bicep to the way I was doing it with a variable for the id: 

{% highlight json %}
connection: {
                name: azureQueueConnectionId
              }
{% endhighlight %}

And there it was! The same error as shown above was thrown

So I finally concluded that the parameter for the **connection name** must be defined in the resource code itself and as a string to make it all work:

{% highlight json %}

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

{% endhighlight %}

