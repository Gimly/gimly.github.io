---
layout: post
title:  "Using Visual Studio Online Build, Squirrel and Azure for continuous integration and deployment of a desktop application."
date:   2015-11-13 10:00:00 +0100
categories: Visual Studio Online, Build, Continuous integration, Squirrel.Windows
---
Introduction
============

I am currently working on a WPF desktop application that is controlling a complex instrument. The software is installed locally on customer's machines and we would like to simplify the application update process as much as possible.

Fortunately, with the relatively new Squirrel.Windows, it has become easy to create an auto-updating application. [Squirrel.Windows](https://github.com/Squirrel/Squirrel.Windows), is an installation and update framework, it puts itself in the same vein as "ClickOnce", meaning a very simple and transparent install process for your application, simply start the setup.exe and it will install and run the app. It has also built-in update capabilities, with the app "calling" a server to see if it has an update and updating itself when needed.

It's a lot more robust than ClickOnce, you can not only publish a .Net application, but also any kind of application. It's also a lot simpler to configure and get running. It's based on Nuget packages, so you can leverage all the NuGet packaging tools and documentation to create your package.

My goal
=======

Basically, I would like to have the build process do the following:

1. Get the latest version of the sources from the source control
2. Update the assembly version to the current build version
3. Build the application
4. Sign the application using a code signing certificate
5. Package the application into a Nuget package for Squirrel
6. Run the Squirrel release creation tool
7. Sign the resulting Setup.exe using the same code signing certificate
8. Upload the results of the creation tools to an Azure Blob storage accessible to customers

The tools we are going to use:

* [Visual Studio Online](https://www.visualstudio.com/products/what-is-visual-studio-online-vs) - It's a "cloud" version of Team Foundation Server, and Microsoft's answer to GitHub and the like. It has everything you need to manage, version, build and test your development project. It's also free for up to 5 developers, even for "closed-source" projects. In that scenario, I'm going to discuss mostly the build capabilities.
* [Squirrel.Windows](https://github.com/Squirrel/Squirrel.Windows) - For installing and updating your application
* PowerShell - For all the scripting needs. VS Online build system allows to run PowerShell scripts during the build process, really useful
* Code signing certificate - Important to remove those annoying messages that the application is not safe during the install
* Azure Blob storage - To publish the application setup. You can probably use something else, like Amazon cloud storage, or any storage where people can then download your application from. Azure is easy because it has build steps directly in VS Online build process.

Prerequisites
=============

I won't dive into the details on how to use the rest of Visual Studio Online, you should already be able to add your code to the source control system of your choice. Personnally, I'm using Git, but this guide should work as well with TFS Source Control.

So, in order to follow this guide, you should already have the code of your WPF/WinForms app pushed to the Visual Studio Online repository.

Procedure
=========

OK, enough talking, let's dive into the complete process. First, open your project on Visual Studio Online, go to the Build tab and hit the green "+" arrow on the left. This will open the *Create new Build Definition* 

![alt text](/images/2015-11-12-vs-online-build-squirrel-windows/Create_New_Build_Definition.jpg "Create New Build Definition Template Selection")

You can either chose to start with a template or start empty. As you can see, it's not only able to build .Net applications using Visual Studio, but also Xamarin and Xcode. Choose the Visual Studio template and click *Next*.

You can then select the settings of on which repository the build will be run, and which branch will be used to do the build. A checkbox at the bottom can be used to define if you want the build to be run at each checkin.

![alt text](/images/2015-11-12-vs-online-build-squirrel-windows/Create_New_Build_Definition_Settings.jpg "Create New Build Definition Settings")

Select the correct repository and branch and click on the *Create* button.

A default build definition is created with the four following steps pre-defined:

 ![alt text](/images/2015-11-12-vs-online-build-squirrel-windows/Default_build_definition.jpg "Default build definition")
 
* **Visual Studio Build** - Relatively self-explanatory, allows you to build your application. It should already be pre-configured and ready to use. It will search for a `.sln` file and start the build process on the contained projects. It also has the ability to *Restore Nuget Packages* automatically.
* **Visual Studio Test** - By default, this step will search all compiled dlls that contain *test* and will try to run unit tests on it. The step will fail if Unit tests have failed, making sure you don't publish things that have non-passing unit tests.
* **Index Sources & Publish Symbols** - This step is useful if you're using *.pdb* files to help you debug your application. It will basically copy all your *.pdb* files to a *symbols folder* so that it can be retrieved and used for debugging sessions.
* **Publish Build Artifacts** - This steps allows you to copy binaries and other files created by the build process on a server of file share so that they can be downloaded. Very useful for debugging and testing the build project, will copy the files that you have selected and allows to be downloaded as a .zip file.

##Get the latest version of the sources from the source control##

At the beginning of the build run, the build engine will automatically pull the latest version of the selected branch and set its *current folder* to the root of your repository content. This means that `./` in scripts will always be the root of your repository.

Therefore, there is nothing you need to do for this, it's done automatically for you.

##Update the assembly version to the current build version##

Each build should get a unique identifier to be easily identified. This meaningful name is called the *build number*. By default, the build number is created using this format:

`$(Date:yyyyMMdd)$(Rev:.r)` which will result in something tht looks like this `20090824.2`

As you can see, it uses flags that are replaced based on the date, and number of build run. You can get a complete list of the available flags [on MSDN](https://msdn.microsoft.com/en-us/library/hh190719.aspx). You can edit the build numer format by going to the *General* tab of the build definition:

 ![alt text](/images/2015-11-12-vs-online-build-squirrel-windows/Build_definition_general_Tab.jpg "Build definition general tab")
 
 When using continuous integration, it is very useful to be able to link back the current version of the application to the build. A nice way to do this is to use the build number as the version number for the application. An easy way to do this, is to create a PowerShell script that will be run before the build process and that will modify the `AssemblyVersion` in the `AssemblyInfo.cs` files contained in your project.
 
 In my solution, I'm using a modified version of a [code sample from a MSDN page](https://msdn.microsoft.com/Library/vs/alm/Build/scripts/index). I have simplified it greatly because I'm using a *trick* to have only one `SharedAssemblyInfo.cs` files that contains the `AssemblyVersion` for all my projects. If you're interested, you can check the how-to for this trick [here](http://weblogs.asp.net/ashishnjain/sharing-assembly-version-across-projects-in-a-solution).
 
<script src="https://gist.github.com/Gimly/9f00a1adb03272b11d6e.js"></script>

To use the PowerShell script during your build process, simply commit it to your source control and push the sources to the server. Then, in the *Build* tab of the definition, click the *Add build step...* button. Select *Utility* and click on the *Add* button for the PowerShell script step.
Then, to configure it, select the step, and add the path and filename to your script in the *Script filename* field.

![alt text](/images/2015-11-12-vs-online-build-squirrel-windows/Add_powershell_task.jpg "Add PowerShell task")

The `AssemblyInfo.cs` files will now be updated before the build, changing the version of your application with the build number. Save your build, and continue to the next step.

## Build the application ##

Not much to do at this step, you can normally leave the *Visual Studio Build* step with its default settings and it should correctly build your application.

![alt text](/images/2015-11-12-vs-online-build-squirrel-windows/Build_application.jpg "Visual Studio Build step")

You can see that, in my case, I have removed the checkbox to *Restore NuGet Packages* in the Build definition, and have added a *NuGet Installer* step at the beginning of the build process. The idea here is that it would allow me to run scripts before the build, but after the NuGet restore. I could, for example, call an utility that would be installed through a NuGet package. 