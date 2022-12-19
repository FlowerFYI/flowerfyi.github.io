---
title: Open sourcing an Azure DevOps extension for Veertu Anka
date: 2021-07-31
categories: [DevOps, Anka]
tags: 
  - Veertu
  - Anka
  - Azure
  - DevOps
  - MacOS
  - build
  - pipeline
  - private cloud
img_path: /assets/img/posts
---
# What is all this about?
In this part of the blog post, we take a brief look at the tools that shall be integrated with each other through the extension implemented in this open source project. Also the reason / need for such an integration is shown and explained. If you're already familiar with Veertu Anka and Azure DevOps and the need of an integration, it's safe to skip this part.. ;)

## What is Veertu Anka?
In the words of the developer Veertu Anka is the following:

> Anka is a suite of software for creating and managing macOS VMs to run on top of Apple hardware and macOS. It's built on top of the native Apple Hypervisor technology for maximum performance. It enables the creation of or integration with existing container based DevOps workflows. Unlike existing macOS virtualization solutions, Anka lets the macOS operating system manage virtual machines in the same manner as other native macOS applications.
{: .prompt-info }

So, Anka is all about providing a container-like environment but for native MacOS scenarios based on native virtual machine technology. A container-like environment in this case means, that you're able to: 
- build a multitude of different virtual machine images with different OS versions and toolstacks installed,
- push all these images into a central registry, containing different tagged versions of them,
- pull these images to different physical MacOS machines running Anka and start the images there.

Because of this setup the availability of Apple hardware only limits the amount of virtual machines you can run at once, but doesn't limit the diversity of toolstacks you can run seamlessly alternating.
This is especially helpful in a company-context, where you want to have managed central builds of your software, because you don't want releases to be built on a developer's machine, but have several different projects all needing installations of different toolstacks or (even worse) the same tools, but in all different versions. Where up to now you had to do parallel installs of different toolchains on the same hardware (leading to unwanted side-effects) or purchase more hardware (going idle when the project currently isn't building), you can now just run different virtual machines with all-isolated toolchains alternatingly using Anka.
If you want to know even more, please check out the website of [Veertu Anka](https://veertu.com/).

## What is Azure DevOps
In the words of the developer Azure DevOps is the following:
> Azure DevOps provides developer services for support teams to plan work, collaborate on code development, and build and deploy applications. Azure DevOps supports a culture and set of processes that bring developers and project managers and contributors together to complete software development. It allows organizations to create and improve products at a faster pace than they can with traditional software development approaches. \\
> [...]\\
> You can use one or more of the following standalone services based on your business needs:
> - Azure Repos provides Git repositories or Team Foundation Version Control (TFVC) for source control of your code. 
> - Azure Pipelines provides build and release services to support continuous integration and delivery of your applications. 
> - Azure Boards delivers a suite of Agile tools to support planning and tracking work, code defects, and issues using Kanban and Scrum methods. 
> - Azure Test Plans provides several tools to test your apps, including manual/exploratory testing and continuous testing. 
> - Azure Artifacts allows teams to share packages such as Maven, npm, NuGet, and more from public and private sources and integrate package sharing into your pipelines.
{: .prompt-info }

So, Azure DevOps is a combination of different services all along the complete application lifecycle management chain, seamlessly integrated into one platform. In our case the service Azure Pipelines is the one of interest, since our target will be to enable the usage of Veertu Anka inside your build- and release-pipelines.
If you want to know even more, please check out the website of [Azure DevOps](https://dev.azure.com/). 

## Why do we need an integration?
While being able to preserve different stacks of toolchains in a registry of virtual machine images and starting them alternatingly on the same piece of hardware is surely a nice thing, to reap the real benefits of this technology we must be able to integrate this functionality into our automated builds! 

Once we've achieved this, we're able to:
- provision exactly the right environment for our build
- start the needed environment in the exact moment we need it for our build
- run our build in this well-defined and preserved environment
- stop and terminate the environment after we're finished with our build, to free the hardware-resources
- define in the build-pipeline of our project the exact needed image to build it, or even different images for each branch of our project

Due to this incredible potential Veertu Anka is of course meant to do this exact thing. Because of this, the developers on Anka already provide plugins for several build-systems (e.g. GitHub, GitLab, Jenkins). But I needed to integrate it into my builds running on Azure DevOps, so I developed an Azure DevOps extension to reproduce the functions which were provided for the other tools tightly integrated into my Azure DevOps pipelines.

# Current state of the extension project
As already mentioned in the introduction, my focus in the development of the extension was to solve my exact problem for the defined environment I was working in. So it comes by no surprise, that the current feature-set of the extension is limited and tailored to this use case. While the extension is already somewhat configurable, it will need some more generalization before it can be released to the public. Also, since there has some time passed since I've developed it, it needs to be adjusted to new features of Veertu Anka and Azure DevOps which are currently not incorporated.
This chapter will shed some light onto the current state of the extension, the steps needed before a release and finally a potential roadmap for the future.

## Current features / usage
After the Azure DevOps extension has been installed into your Azure DevOps organization you gain three new elements to integrate Veertu Anka into your pipelines. A Service Connection to define the properties needed to reach your Anka Controller and the two actual build-tasks, which allow you to start and terminate Anka instances from within your pipeline. 

All these elements are currently tailored to support an Anka environment, which
- is running on-premises
- has an Anka Controller which can be accessed from a self-hosted (on-premises) build agent without authentication
- can access the internet from within the running Anka instances either directly or through a proxy without authentication

And an Azure DevOps environment which
- has a self-hosted agent defined on which the start / terminate tasks can run and from which the Anka Controller can be accessed within the on-premises network
- has an Agent Pool for self-hosted agents defined, which will hold the automatically generated and deleted agents for the Anka instances
- has a generated Personal Access Token with the rights to manage the Anka Agent Pool to dynamically add and delete the Anka DevOps Agents

### Anka Controller Service Connection
Through Service Connections in Azure DevOps it's possible to provide the build-tasks inside a project with the necessary configuration data to access a resource without having to configure these parameters for each task.
For this purpose the extension defines a new Service Connection, which can be created using the Service Connection administration:

![Image of the selection of the Anka Controller in the Service Connection administration](anka_devops_new_anka_controller.png)
_Anka Controller connection_

After starting the creation, the following fields can / must be filled (as seen in the image below):
- **User for Agent**: While Anka assumes that all processes run under an automatically logged in administrative user, the extension expects that the Azure DevOps agent should be run under a separate less privileged user. This user must be defined here and the logged in Anka user must have the right to `sudo` to this user without entering a password.
- **Path to script**: The current implementation of the extension expects all Anka images to contain a shell-script which will download, install and start the Azure DevOps Agent inside the instance. This parameter is for setting the path to this script.
- **Install-Path for Agent**: The destination path, where to install the downloaded agent inside the Anka instance.
- **Path to Agent-Logs**: Path to logs for the agent and build inside the Anka instance, this logs can be accessed through Anka functions like `anka run` while you're logged in on the Anka build node running the image.
- **Proxy for Agent-Connection**: If the Azure DevOps Agent needs to use a proxy to access the internet from within the Anka instance, this proxy can be configured here.
- **Agentpool for VMs**: The Self-Hosted Agent Pool defined inside Azure DevOps to hold the agents started within the Anka instances.
- **PAT for Agentpool**: The Personal Access Token with the permissions to manage the Agents inside the *Agentpool for VMs*.

![Image of the settings for the Anka Controller connection](anka_devops_anka_controller.png)
_Anka Controller connection settings_

### Anka Build-Tasks
The two new Anka tasks provided by the extension will show up in your build task list as shown here:

![Anka tasks available through the extension](anka_devops_tasks.png)
_Anka tasks_

Since you need to run these tasks within a self-hosted agent with access to the Anka Controller (while your actual build should run inside the Anka instance) you'll most likely end up with a pipeline-definition looking like this (or the YAML-equivalent of it):

![Example of a pipeline configuration with included Anka tasks](anka_devops_pipeline.png)
_Pipeline example_

As shown in this image, you have three different jobs running on different agents:
- The first job runs on the self-hosted agent to run the "Anka Start Instance" task.
- The second job runs on the Anka Agent Pool inside the Anka instance which has been started inside the first job.
- The last job runs on the self-hosted agent again to run the "Anka Terminate Instance" task. **Important**: This job and task must be configured to run always, even in the case of any errors or a user cancelling the build. If you miss out on this configuration, you'll start collecting running Anka instances until all your nodes are full... ;)

The "Anka Start Instance" task provides the option to set a reference to the Anka Controller Service Connection which has been defined, the Anka VM-Name from the Anka Registry which should be started and its tag. Additionally an Anka-Agent-Identifier must be set, which has to be unique for this build (in this example by using the BuildId in the identifier). Through this identifier (which is set as a special capability on the newly registered DevOps agent) the build-job can identify this exact Anka instance to run on.

![Image of the settings for the Anka start task](anka_devops_start.png)
_Anka start task settings_

The "Anka Terminate Instance" task also provides the option to set a reference to the Anka Controller Service Connection and needs the exact same Anka-Agent-Identifier set by the "Anka Start Instance" task, to find the right Anka instance to terminate.

![Image of the settings for the Anka terminate task](anka_devops_terminate.png)
_Anka terminate task settings_

## Current architecture / flow
Now that we know / have learned which parts are provided by the Azure DevOps extension, we can take a look at the actual architecture and flow.

The following diagram shows what happens behind the scenes when the example pipeline-flow (mentioned above) is actually running:

![Current architecture flow as image showing the components](anka_devops_architecture.png)
_Current architecture / flow_

Here's the description of the steps (as numbered in the diagram):
1. The build pipeline runs the "Anka Start Instance" task on a self-hosted agent inside the on-premises network.
2. The "Anka Start Instance" tasks contacts the Anka Controller and tells it to start an instance of the Anka VM-image configured in the task.
3. The Anka Controller triggers the start of the instance on an available Anka build node.
4. The startup-script inside the newly created Anka instance downloads and installs the Azure DevOps agent, which is newly registered in the configured Anka Agent Pool in Azure DevOps and which gets the Anka-Agent-Identifier set as a capability, to be identified later.
5. The build-pipeline finds the newly available build-agent inside the Anka Agent Pool through the Anka-Agent-Identifier.
6. The actual build tasks run inside the Anka instance to build the project.
7. The last job of the pipeline triggers the "Anka Terminate Instance" task on a self-hosted agent inside the on-premises network.
8. The "Anka Terminate Instance" task deletes the no longer needed agent from the Anka Agent Pool.
9. The "Anka Terminate Instance" tasks contacts the Anka Controller and tells it to terminate the instance which it has identified through the Anka-Agent-Identifier.
10. The Anka Controller triggers the termination of the instance.

# Work to be done before initial release
So, here's the initial (most likely incomplete) list of things which I think need to be addressed / fixed before I can release a first version of the Azure DevOps extension for Veertu Anka.

| Topic | Description |
| ----------- | ----------- |
| Veertu Anka API check | Check the Anka APIs for any noteworthy updates, which should be incorporated either into the first release or into the roadmap. |
| Azure DevOps SDK check | Check Azure DevOps SDK for noteworthy updates, which should be incorporated either into the first release or into the roadmap. Especially checking the handling of build variables to reduce overhead for managing the Anka-Agent-Identifier and the Anka instance id of the started instance. |
| Move of Startup Script | As described, currently the extension expects all Anka images to contain a shell-script which will download, install and start the Azure DevOps Agent inside the instance. This function introduces the need to build new Anka images whenever something in the script needs to change. This script has definitely to be moved into the extension, to be handled internally. Optimally this should be implemented by defining a default-script inside the extension, which then can be overwritten by a custom script in either the Anka Controller connection or the Anka Start instance task. |
| Unit Tests | Implement unit tests for all functions of the extension. |

# Ideas for future features / roadmap
And finally some ideas, what can be incorporated into future releases.

| Topic | Description |
| ----------- | ----------- |
| Anka Controller Authentication | As described above, currently the extension assumes that the Anka Controller can be accessed from a self-hosted (on-premises) build agent without authentication. This should be changed to provide the possibility of an authentication at the Anka Controller. |
| Agentless task | Currently the extension assumes that all Anka tasks must be run in a self-hosted agent on-premises to gain access to the Anka Controller. But maybe somebody has a deployment of an Anka Controller with a public endpoint reachable from the internet. In this case the task could be run directly inside Azure DevOps without utilizing an agent at all. For this purpose a version of the task should be provided as an agentless task. |
| More comfort in task-configuration | Currently the Anka image name and tag have to be entered manually into a text-field. It would be great to have a selection list here, of course this could only apply to the agentless task, since the entries for the selection list would have to be available on a public endpoint reachable from the internet |
