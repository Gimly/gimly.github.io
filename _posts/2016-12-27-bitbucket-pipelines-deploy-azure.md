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

We can now try to run the application and check that everything is working as expected.

```
dotnet run
```

This will start kestrel and you should be able to access the application by going to 
<http://localhost:5000>.

Check that everything is working correctly, check that when you access the counter component
pushing the "increment" button, the counter increses. Also check that the fetch data component correctly
displays the weather data.

Step 2: Create the repository in Bitbucket
------------------------------------------
We'll create a simple repository in Bitbucket where we will store the project's code and where we will configure
Pipelines.

First, login to your Bitbucket account and go to *Repositories* > *Create repository*.

Use the following options:

- **Repository name**: NetCorePipelinesToAzure
- **Repository type**: Git

![Create repository](/images/2016-12-27-bitbucket-pipelines-deploy-azure/CreateNewRepository.jpg)

Next, we'll push the local Git repository that we created in the first step to the freshly created repository in Bitbucket.

```
git remote add origin ssh://git@bitbucket.org/Gimly/netcorepipelinestoazure.git
git push -u origin master
```

Once everything was pushed correctly, check that your source was correctly pushed by going to the *Source* option in Bitbucket.

![Bitbucket source view]("/images/2016-12-27-bitbucket-pipelines-deploy-azure/BitbucketSourceView.jpg")

Step 3: Create Azure App Services Web App
-----------------------------------------
Final preparation step, we need to create an Azure App Services Web App that will host our application.

Login to your Azure account and select *New* > *Web + Mobile* > *Web App*.

Setup your Web App by chosing a name, a resource group and selecting an *App Service plan* (the free tier is fine for testing).

Click create and wait for the end of deployment.

Since we're going to deploy using FTP from Bitbucket, we'll setup a *FTP/deployment username* and *password*. This needs to be
configured by going to 
