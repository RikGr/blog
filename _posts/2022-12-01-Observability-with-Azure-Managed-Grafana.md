---
layout: post
title: Observability with Azure Managed Grafana
author: Rik Groenewoud
tags: microsoft azure observability grafana
---

During my visit to Monitorama 2022 in Portland it became clear to me that the phenomenon of "Observability" is here to stay. 
The people from Honeycomb.io  even wrote an O'Reilly book on the subject called: [Observability Engineering](https://info.honeycomb.io/observability-engineering-oreilly-book-2022). 

My initial sceptics  made place for enthusiasm. After I read the book and learned more on what is meant by Observability, my initial skepticism ("This must be the new buzz word for plain monitoring!") made place for enthusiasm. I truly believe it can help it can help DevOps Engineers running their distributed operations in the cloud. 

So when the call for papers came for our bi-annual XPRT magazine, I was eager to write an article on this subject. Together with my good colleague [Casper Dijkstra](https://xpirit.com/team/casper-dijkstra/) we decided to write on Observability theory and to bring this into practice with Azure Managed Grafana. 

## TLDR version of the article

**Observability**

This is the definition used in the book "Observability Engineering": 

*“... our definition of “observability” for software systems is a measure of how well you can understand and explain any state your system can get into no matter how novel or bizarre”*

**Azure and Observability**

Microsoft embraces the idea of Observability. The advocate the use of Log Analytics Workspaces to bring all logging and metric data together. 
For visualization of this data Azure has basic dashboarding functionality. We argue that Grafana can be a great addition to this.
With the new Azure Managed Grafana offering 
The most important assets are: 

- Usability 
- Advanced dashboarding visuals
- Bring together multiple data sources, also from outside of Azure (i.e. Azure DevOps)
- Not only vizualize but also dive into the data by quering realtime logging and metrics
- Large community with lots of dashboard templates 
- Integrate dashboarding in your DevOps way of working by deploying dashboards as code using CI/CD tooling 

We then show a basic setup of how to run dashboards-as-code using a DevOps Pipeline and tell more about the cost aspect of this offering

**Conlusion**

By bringing all these data sources together and combining them in a smart and useful way in Azure Managed Grafana, observability becomes something real and will add value to the existing tools that Azure offers. In the current age of highly complex distributed systems it is no longer about only trying to prevent issues from occurring, but to make sure engineers have the right tools to literally observe what is going on and to locate the issues as quickly as possible. 

## Full article and more
[Link](https://xpirit.com/wp-content/uploads/2022/10/Xpirit_XPRT_magazine_13_final.pdf?utm_campaign=Xpirit%20-%20Magazine%2013&utm_source=download-page) to the magazine with our article from page 45.



