---
layout: post
title:  Leveraging Azure Monitor Baseline Alerts
author: Rik Groenewoud
tags: microsoft azure observability alerts monitoring
---

## Intro

In the wonderful world of Microsoft Azure things are often easier said than done. Or to rephrase it a little bit: Often, things are easy in the beginning, but in the blink of an eye it is more complex than you could have ever imagined.
A prime example of this is **managing alerts** for your resources. The creation of an alert rule itself is easy and for obvious metrics such as CPU or RAM it is not so difficult to come up with some thresholds and severity levels.

But where to place alerts when we are managing and maintaining a large landscape of Azure Infrastructure such as an Enterprise Landing Zone? What are good thresholds and severity levels for different kinds of resources and alert types? How should we warn people when an alert is triggered? Who should be alerted? How to prevent a complete flood of alerts cause alert fatigue? These are the questions that are not so easy to answer.

Luckily, Microsoft together with the community, has been working on a solution for this: **Azure Monitor Baseline Alerts (AMBA)**. I will go into more detail on what this baseline is and how it can help you setup a proper set of alerts for your Azure environment.

## What is AMBA?

As this [official documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alert-options#azure-monitor-baseline-alerts-amba) states:

*AMBA is a central repository that combines product group and field experience driven alert definitions that allow customers and partners to improve their observability experience through the adoption of Azure Monitor.*

and:

*AMBA has patterns that group alerts from different resource types to address specific scenarios. Azure landing zone (ALZ), which is also suitable for non-ALZ aligned customers, is a pattern of AMBA that collates platform alerts into a deployable at-scale solution.*

What this means is that for a landing zone scenario AMBA comes with an opinionated set of predefined alerts on the most common resources deployed in landing zones. This makes it a powerfull tool to get a comprehensive set of alerts in place quickly. The alerts are based on best practices and are created by the Azure product group and field experience.

To summarize, it a great starting point and enabler to improve landing zone observability quickly. Because it is al parameterized, it is also easy to customize to your own needs and to tailor it for customer specific situations.

An important feature of AMBA is that is **policy-driven**. Instead of creating the alert resources themselves it consists policies to create the alerts on the specific resources if they are missing. Policy Intiatives are used to logically group the alert definitions together. With Policy Assignments you can assign the initiative to the preferred scope such as Management Groups or subscriptions.

The policies not only provide for creating alerts but also for creating **processing rules**. Processing rules are used to route the alerts to the right people or systems. This is done by creating Action Groups and linking them to the alerts. Within the action group you can choose to, for example send out an e-mail, SMS and/or to trigger a webhook.
So not only the alerts itself are covered but also the process of warning people when an alert is triggered is covered.

A great milestone for the AMBA initiative was the incorporation into the official Microsoft documentation. This was a big deal for the maintainers as this LinkedIn posts shows:

[![image](/images/blog-8.1.png)](https://www.linkedin.com/posts/bruno-gabrielli-4992528_create-alert-rules-for-an-azure-resource-activity-7166385933951352833-Dic3?utm_source=combined_share_message&utm_medium=member_desktop)

## How to deploy?

In the AMBA documentation there is a [deployment guide](https://azure.github.io/azure-monitor-baseline-alerts/patterns/alz/deploy/Introduction-to-deploying-the-ALZ-Pattern/) which gives example methods to deploy the alerts for the ALZ scenario.

It describes the following methods:

- Azure CLI
- Azure Powershell
- Azure Pipelines
- GitHub Actions

The repository comes with all the necessary code and templates to deploy the alerts. The deployment guide is clear and easy to follow. For our own landing zone however, the preferred Infrastructure as Code (IaC) tool is **Terraform**. Could I use Terraform to deploy the AMBA alerts?

## Transition to Terraform

Since there was no out-of-the-box support for Terraform we needed to set this up ourselves.
What we did :

1. Deploy the policies to an demo Azure environment using the Powershell method from the above mentioned documentation. Then copy all policy definition JSONs as separate policy definition files in our own repository. This step was needed because our Terraform code expected the policy definitions JSON in the same syntax as of a deployed policy definition.
2. Re-create the policy set definitions (initiatives) into our own format. We created the following definition structure to cover all aspects of our landing zone. This structure is the same as AMBA itself uses:
    - Connectivity
    - Identity
    - Landing zone (meaning workload spokes)
    - Management
    - Service Health
3. Create Assignments of the initiatives to the preferred scope (Management Group in our case)

## Fine-tuning

After the deployment of the AMBA alerts we started to receive alerts. This was a good thing, but we quickly noticed that some alerts were not relevant for our environment. For example, we received unavailable alerts for VMs that used auto-shutdown during non-office hours. The good thing is since we have the policy definitions defined per environment, we can easily tweak and adjust the alerts to the specific needs of the customer. For us the best approach was to first start with defaults provided by the AMBA and run with this for a few weeks. Over time, adjust the alerts thresholds and configurations to the specific circumstances of the environment.

## Conclusion

When you ask me, the AMBA initiative is a great new option in the world of Azure monitoring. It provides a great starting point for setting up alerts in your Azure environment at scale very quickly. It is policy-driven and easy to deploy. The fact that it is parameterized makes it easy to adjust to your own needs. The only downside is that it is not yet supported by Terraform. But with some effort, you can set this up yourself.

## References

- [Azure Monitor Baseline Alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alert-options#azure-monitor-baseline-alerts-amba)
- [Deployment Guide](https://azure.github.io/azure-monitor-baseline-alerts/patterns/alz/deploy/Introduction-to-deploying-the-ALZ-Pattern/)

<script src="https://giscus.app/client.js"
        data-repo="RikGr/cloudwoud"
        data-repo-id="R_kgDOHLlC9w"
        data-category="Announcements"
        data-category-id="DIC_kwDOHLlC984CO_2O"
        data-mapping="pathname"
        data-reactions-enabled="0"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="light"
        data-lang="en"
        crossorigin="anonymous"
        async>
</script>
