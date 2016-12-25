---
layout: post
title:  "Continuous deployment of ASP.Net Core to Azure using Bitbucket Pipelines."
date:   2016-12-27 10:00:00 +0100
categories: Azure
comments: True
author: Xavier Hahn
tags:
- Continuous deployment
- Bitbucket
- Azure
- Pipelines
---
Continuous deployment of ASP.Net Core to Azure using Bitbucket Pipelines.
====================================
Bitbucket Pipelines is the new continuous delivery feature from Bitbucket.
It is meant as a replacement for Bamboo cloud that will be discountinued by
the end of January 2017.

It's built over Docker, it basically loads a Docker image (which you can choose) and allows you to
run a script to build and deploy your application.

The goal of this article is to guide you through the process of deploying and
ASP.Net Core application to Azure App Services using Bitbucket Pipelines.

Pre-requisites
--------------
To follow this tutorial, you'll have to have installed both NPM and the latest .Net Core (1.1) runtimes.

* To install NPM, you'll have to install NodeJS. Follow the instructions on the [official website](https://nodejs.org/en/).
* For ASP.Net Core, follow the instructions ont the [official website](https://www.microsoft.com/net/core#windowsvs2015). Chose the 
  command line installation for your OS of choice. 

Step 1: Creating the ASP.Net Core application
---------------------------------------------
To get started, let's create a simple ASP.Net Core application. I'll be using the 
Yeoman ASP.Net Core Single Page App generator. The first step is therefore to install
yeoman and this generator.

Both can be installed easily through NPM:

```
npm install -g yo generator-aspnetcore-spa
```

Once installed, create a folder where you want to create your project, initialize a git repository 
and run the command to execute the generator. The options you select for the generator doesn't matter,
I've used "Angular 2", yes to unit tests and the default for the project name.

Finally, we'll do a quick commit for good measures:

```
mkdir dotnet-azure-pipelines
cd dotnet-azure-pipelines
git init
yo aspnetcore-spa
git add --all
git commit -m "Run Asp.Net Core SPA generator"
```

We can now run 