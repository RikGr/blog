---
layout: post
title: Control an IoT device based on Microsoft Graph API data. Azure Logic Apps and Azure Functions make it easy! 
author: Rik Groenewoud
tags: microsoft azure iot logicapps azurefunctions powershell
---

## Intro
One of latest additions of spaces we deliver solutions for at Xpirit is the rather broad subject of "Internet of Things" also known as IoT. We call it Smart & Connected services. The idea is to accelerate the IoT journey of our customers. To make all employees familiar and enthusiastic about this subject, everyone got a IoT LED light called "Blinky". We used it to play around and do workshops with it to learn what Azure has to offer when it comes to IoT.

![Blinky](/images/blog-3.1.jpg)
## My idea
Inspired by all kinds of cool use cases from my colleagues such as: 

 - make the LEDs go pulsing like Kit from Knight Rider
 - turn Blinky on when you are in a live Teams call 
 - Change the color of Blinky based on the room temperature
 - etc. 
 
I also came up with an idea of my own.What if could get a warning signal when I have an appointment in my Outlook Calendar? 
Since I do not always have Outlook open, there is always the risk of missing a Teams call of any other appointment. An extra visual warning would be a great help!
My idea was to give Blinky a pulsing red light when an appointment is within 5 minutes of its scheduled start time.
## Sketch and outline of the solution 
I came up with the following infrastructure to make this happen: 

![sketch](/images/blog-3.2.png)

- I use a Logic App to authenticate with my Outlook Calendar in an easy way and retrieve all my appointments via the Office 365 Outlook "Get Events" action. 
- Next, I put the retrieved JSON with all my appointments to a storage account. 
- Finally the logic app sends a POST request to the Function App 1 to let it know it can grab new data from the storage account. 

![logic app](/images/blog-3.3.png)

The next step is to pick up this raw data and do some calculations on it. This happens in the Function App 1. Using Powershell it picks up the data from the storage account and see if there are appointments within 5 minutes. If so, it sends a POST request out to the second Function App. 
Below you see the script of Function App 1 (the script that I actually wrote myself):

```powershell
using namespace System.Net
# Input bindings are passed in via param block.
#param($request, $TriggerMetadata)
param(
    $request
)

#Variables
$container = 'calendarevents'
$blob = 'events.json'
$storageaccountname = 'rgautomationeuwebaac' 

# Write to the Azure Functions log stream.
Write-Host "The CheckCalenderEvents function received a request: $request."

#Get storage account key 
$keyvault = 'kv-automation-euwe' 
$secret = Get-AzKeyVaultSecret -VaultName $keyvault -Name "storagekey" -AsPlainText

#Get data from storage account
$context = New-AzStorageContext -StorageAccountName $storageaccountname -StorageAccountKey $secret
Get-AzStorageBlobContent -Container $container -Blob $blob -Destination ./events.json -Context $context -Force
$data = Get-Content .\events.json | ConvertFrom-Json
$data = $data.value

# Filter for future events
$result = foreach ($d in $data) {
    $today = Get-Date -AsUTC
    $d.start | Where-Object { [datetime]$_.DateTime -GE $today }

}
Write-host "Result: $result"

#Calculate if there are events within 5 minutes. If so change state of Blinky to red pulse, if not so turn off Blinky
foreach ($r in $result) {
    $t = $today - [datetime]$r.DateTime
    Write-Host $t

    if ($t.TotalMinutes -GT -5) {
        $body = '{
            "deviceId": "bare-black-canary",
            "mode": 2,
            "brightness": 255,
            "saturation": 255,
            "hue": 7
        }'
        Invoke-WebRequest -Uri 'https://func-updatetwin.azurewebsites.net/api/' -Body $body -Method PUT
        break
    }

    else {
        $body = '{
            "deviceId": "bare-black-canary",
            "mode": 0,
            "brightness": 255,
            "saturation": 255,
            "hue": 7
        }'
        Invoke-WebRequest -Uri 'https://func-updatetwin.azurewebsites.net/api/' -Body $body -Method PUT
    }
}

# Associate values to output bindings by calling 'Push-OutputBinding'.
Push-OutputBinding -Name Response -Value ([HttpResponseContext]@{
        StatusCode = [HttpStatusCode]::OK
        Body       = $body
    })
```
Lastly, the Function App 2 contains the code that tells Blinky what to do. Based on the posted request it sends a POST request for a pulsing red light, or it turns Blinky into mode 0 which effectively turns the light off.
So the actual control of the IoT device is in Function App 2 it sends the mentioned POST request to Azure IoT Hub where my Blinky is registered and from there it sends the signal towards the physical device itself (using wifi). 

A little bit on the hardware of Blinky. The heart of the device is a [ESP8266 NodeMCU](https://randomnerdtutorials.com/projects-esp8266/) and to control it we use [Arduino](https://www.arduino.cc/en/software). Please follow the links if you want to know more.
## Conclusion
What I think is cool about this project, is that with some easy-to-use tools like the Logic- and Function Apps, I made my plan a reality with no advanced coding skills required. When I started thinking about how this should work, I was struggling with how to authenticate to my personal Calendar and thought about writing one big script to do all the work. The moment I took a step back and looked at what Azure had to offer out-of-the-box, it turned out I could do this simpler, secure and surprisingly stable as well: since this application is live it kept on running ever since!

To me this is the power of the low code solutions within the Azure ecosystem: Let Azure do the work for you and make powerful cloud computing solutions accessible for everybody.