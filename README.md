A simple node app with Azure Pipelines and App Service
======================================================
This pipeline is based off of [a sample node app by Ed Thomson](https://github.com/calculator-demo/calculator) that lets you do basic math operations in a browser-based calculator. All the math happens on the server, so we've been referring to it as 1+1... as-a-service. The goal of this is to provide some familiarity with Azure Pipelines YAML concepts and functionality so you can get get off and running quickly.

[![Build Status](https://dev.azure.com/calculator-sample/calculator-sample/_apis/build/status/matthewisabel.calc-stage-sample?branchName=master)](https://dev.azure.com/calculator-sample/calculator-sample/_build/latest?definitionId=2&branchName=master)

## Forking and running locally
1. Clone the repo
2. `npm install`
3. `npm start`
4. View your calculator in action at `localhost:3000`

## Getting started with Azure Pipelines now...
First, [install the Azure Pipelines app on the GitHub marketplace](https://github.com/marketplace/azure-pipelines) (it's free!). As part of the install, you'll be guided through creating an account and then you'll land in the setup wizard to get started with your first pipeline. If you're starting with this repo, it'll auto-detect this is a node project and you can pick the first sample template that looks like this below:

```yaml
# Node.js
# Build a general Node.js project with npm.
# Add steps that analyze code, save build artifacts, deploy, and more:
# https://docs.microsoft.com/azure/devops/pipelines/languages/javascript

trigger:
- master

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: NodeTool@0
  inputs:
    versionSpec: '10.x'
  displayName: 'Install Node.js'

- script: |
    npm install
    npm run build
  displayName: 'npm install and build'
```

## Let's talk YAML
So let's talk about what this means section-by-section (and you can also read along in our [YAML schema concepts](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema)
  
```yaml
trigger:
- master
```
First, this means that we're only listening for changes to master, whether it be PRs or new commits. [There are other triggers available](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/triggers?view=azure-devops&tabs=yaml), but this one is just for master. Moving along...

```yaml
pool:
  vmImage: 'ubuntu-latest'
```

Okay, so this means we're using the latest Ubuntu pool. Right now that image is Ubuntu-16.04. While this is recommended for Node, [there are certainly other hosted pools available](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops#default-agent-pools). Also, if you're curious what it means to be using Ubuntu 16.04, [you can see everything installed on the image here](https://github.com/Microsoft/azure-pipelines-image-generation/blob/master/images/linux/Ubuntu1604-README.md).

```yaml
steps:
- task: NodeTool@0
  inputs:
    versionSpec: '10.x'
  displayName: 'Install Node.js'

- script: |
    npm install
    npm run build
  displayName: 'npm install and build'
```

And finally, on to the steps, which are the actual code that's executing once we're running our pipeline. For now, all you need to know is that steps run sequentially, and what they're running can be as simple as a script with a single line or it can be more complex, like a task. We'll start with a task first to install node and then a few script commands to generate our build.

So first, this is using NodeTool to install Node version 10. So what's NodeTool@0 and why is it a task? [Tasks are the fundamental building blocks of pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/tasks?view=azure-devops&tabs=yaml), and this one, NodeTool, is @0 which means version 0. NodeTool, like many other Azure Pipelines tasks is [open source on GitHub so you can see exactly what it does](https://github.com/Microsoft/azure-pipelines-tasks/tree/master/Tasks/NodeToolV0). We can change this and lock in to specific version, but let's assume 10 works for us.

Next, we're on to the script step which will run sequentially after the node install. All we're doing out of the gate is an `npm install` and `npm run build` to get our packaged artifact. That's a pretty basic node pipeline, but let's add a single line. Our sample code here actually has tests, so as part of that script let's also run our tests. Not only will we get these results as part of our build, but Azure Pipelines will visualize them in the product as well and help us track test failures.

```diff
 - script: |
     npm install
     npm run build
+    npm run test
   displayName: 'npm install and build'
```

And there we have it. For reference, [here's the two commits that introduce this YAML](https://github.com/matthewisabel/calc-stage-sample/compare/d4f2488..0f016ad). [And here's the build result in Azure Pipelines](https://dev.azure.com/calculator-sample/calculator-sample/_build/results?buildId=3) where we can see the successful run.

## CI *and* CD with multi-stage
We're in good shape now we can save and run this basic pipeline. While we're building our code in this first step, we really want to deploy our calculator and get it up to app service. What we're going to do next is add a second stage to our pipeline to handle deployment. In the advanced section later on we'll talk about how we'll be able to use stages as boundaries for checks, like approvals, which are coming soon to Azure Pipelines and will allow us to control the flow of our code from staging to production. For now though, let's keep it simple with two stages.

So let's ramp back up on what it means to have stages and how it changes what we have. Compared to our basic setup, let's look at an example of how ours will look with stages:

- Calculator Multi-Stage (pipeline)
  - Build (stage 1)
  - Deploy (stage 2)
  
Or in the stage graph format you'll see in Azure Pipelines:

<img width="528" alt="Build-deploy graph" src="https://user-images.githubusercontent.com/2132776/57195356-7474ed00-6f1f-11e9-8317-52b9321dd9e7.png">

This is pretty basic, but we're actually going to introduce another new concept, job. A job is the container that our steps live in, and can be executed sequentially or in parallel. Even our simple pipeline above actually had both a stage and a job, but we didn't need to specify it until we moved on to a more complex pipeline with explicit stages. So let's see a high-level overview of our future YAML structure with the job stage and job containers.

- Pipeline [Calculator Multi-Stage]
  - Stage [Build] 
    - Job [Build and Test]
      - Step [Install node]
      - Step [Run scripts]
      - Step [Archive files to zip artifact]
      - Step [Publish build artifact]
  - Stage [Deploy]
    - Job [Deploy to prod]
      - Step [Download build artifact]
      - Step [Publish to Azure App Service]

[Jobs have a lot of features we can associate in YAML](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/phases?view=azure-devops&tabs=yaml), and while we're keeping this sample simple, you could imagine wanting to target an app to run across multiple platforms. Take [GitHub desktop](https://dev.azure.com/github/Desktop/_build/latest?definitionId=3) for example, which needs to support mac, Linux, and Windows. The GitHub team can easily do this by defining a matrix that runs jobs in parallel to make sure our CI passes across those platforms.

In our example, we're just building on our simple pipeline, but you'll also notice a few new steps. Since we've split across stages that have jobs that use their own new agents, we have one job with steps which zips and publishes our build artifact in the first stage, and another job that downloads the build artifact and then publishes it as steps.

So let's get into how we do this in YAML. First, we can use the Azure Pipelines YAML editor to make this a little smoother. The nice thing about the YAML editor is it lets us easily add tasks we need, like archiving our build, publishing the build artifact, downloading the published artifact, and finally publishing that artifact to Azure App Service.

<img width="1440" alt="Azure Pipelines YAML editor" src="https://user-images.githubusercontent.com/2132776/57201353-e1ab7100-6f65-11e9-98ef-b03c3926b98b.png">

I used the YAML editor to author this using by searching for the appropriate tasks and [you can see the change commit here and the resulting YAML file](https://github.com/matthewisabel/calc-stage-sample/commit/470d1897023b123fe6a2d99ab16a323d4ba03ac1). If you want to build up this file yourself, search for these tasks in the YAML editor to get going quickly, but I'm going to skip over that and go straight to discussing the changes. We'll start by looking at the whole YAML file here and [the successful build result first](https://dev.azure.com/calculator-sample/calculator-sample/_build/results?buildId=4) before running through it piece-by-piece.

```yaml
trigger:
- master

pr: none

variables: 
  # Service connection as configured in project settings
  prodConnection: 'isabelProd'

  # App name
  prodApp: 'isabelProd'

stages:
- stage: build
  displayName: 'Build'
  jobs:
  - job: build_test
    displayName: 'Build and test'
    pool:
      vmImage: 'Ubuntu-16.04'
    steps:
    - task: NodeTool@0
      inputs:
        versionSpec: '10.x'
      displayName: 'Install Node.js'
    - script: |
        npm install
        npm run build
        npm run test
      displayName: 'npm install, build, and test'
    - task: ArchiveFiles@2
      inputs:
        rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
        includeRootFolder: false
    - task: PublishPipelineArtifact@0
      inputs:
        targetPath: '$(System.ArtifactsDirectory)'


- stage: prod
  displayName: 'Prod'
  dependsOn: build
  jobs:
  - deployment: deployProd
    displayName: 'Deploy prod'
    environment: isabelProdEnv
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@1
            inputs:
              downloadPath: '$(System.DefaultWorkingDirectory)'
          - task: AzureWebApp@1
            inputs:
              azureSubscription: $(prodConnection)
              appType: 'webAppLinux'
              appName: $(prodApp)
              runtimeStack: 'NODE|10.1'
              startUpCommand: 'npm run start'
```

A few small changes at the top, the main one is just for this demo we're not running this pipeline on PRs, so you'll see `pr: none`. We could do more with conditional stages to have control flow that's more targeted, but we'll cover that in the advanced section. The first addition you may notice is that I've defined some variables. As part of this before authoring the YAML, I've introduced my App Service connection. [Here's a simple guide to configuring a service connection](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops), but I'll be referencing it in my YAML via variables the rest of the way. The service connection lets us reference in YAML our Azure App Service target we'll be deploying to in our second stage.


```yaml
variables: 
  # Service connection as configured in project settings
  prodConnection: 'isabelProd'

  # App name
  prodApp: 'isabelProd'
```

From there, we're onto our first build stage.

```yaml
stages:
- stage: build
  displayName: 'Build'
  jobs:
  - job: build_test
    displayName: 'Build and test'
    pool:
      vmImage: 'Ubuntu-16.04'
    steps:
```

I'm leaving out the steps for now, but you can see the structure of stages, which contain each stage, which contains jobs, and each job which contains a series of steps.

```YAML
    steps:
    - task: NodeTool@0
      inputs:
        versionSpec: '10.x'
      displayName: 'Install Node.js'
    - script: |
        npm install
        npm run build
        npm run test
      displayName: 'npm install, build, and test'
    - task: ArchiveFiles@2
      inputs:
        rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
        includeRootFolder: false
    - task: PublishPipelineArtifact@0
      inputs:
        targetPath: '$(System.ArtifactsDirectory)'
```

The goal here is just to take the build from the default working directory and publish it to the Artifacts directory. If I'm looking at this pipeline run later in the Azure Pipelines UI, I can see this artifact we're publishing and download it if I want to dig in and debug.

Now let's move to the simple deployment stage. There are a few concepts here that are totally new that we should dig into.

```yaml
- stage: prod
  displayName: 'Prod'
  dependsOn: build
  jobs:
  - deployment: deployProd
    displayName: 'Deploy prod'
    environment: isabelProdEnv
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@1
            inputs:
              downloadPath: '$(System.DefaultWorkingDirectory)'
          - task: AzureWebApp@1
            inputs:
              azureSubscription: $(prodConnection)
              appType: 'webAppLinux'
              appName: $(prodApp)
              runtimeStack: 'NODE|10.1'
              startUpCommand: 'npm run start'
```

The first thing you may notice that this job has a different treatment in that it's specifically a `deployment` job. This lets us use a few great features, like configuring an `environment`. In the above YAML we just named our environment `isabelProdEnv` which I can do without any other setup, once the string is there it's automatically created for you in Azure Pipelines. Environments are a new feature that let us have full end-to-end traceability of commits and work items and let us jump back and look at our deployment history. You can also associate resources to unlock even more advanced features for k8s, but we're going to keep it simple here. Once it's defined with a string in YAML, it's automatically created and we're off and running.

Next, you'll notice after environments we can see there's a `strategy` defined. This is another exciting CD feature where right now we're just declaring a simple `runOnce` pipeline, but in the future we'll be able to say we want to do a blue-green, rolling, or canary deployment. For our straightforward deployment, `runOnce` works just fine.

After our deployment information, you can see our steps are defined like they are anywhere else in YAML. In this case we're downloading the artifact we created in the first stage and then using the AzureWebApp@1 task to upload the zip and run it with a simple `npm run start`. And with this simple pipeline, we have our code building and deploying to https://isabelprod.azurewebsites.net/.

And there we have it with our code published and you can see [our pipeline run information inside of Azure Pipelines](https://dev.azure.com/calculator-sample/calculator-sample/_build/results?buildId=4).

<img width="1436" alt="Azure Pipelines summary" src="https://user-images.githubusercontent.com/2132776/57254600-ae2d1d00-701f-11e9-9abe-b57f17b463ae.png">

From our summary, we can jump in and check out the logs from our run. Since both our build and deploy stages succeeded there isn't too much interesting to dig through here, but you can see it's easy to navigate between logs of different stages.

<img width="1439" alt="Azure Pipelines logs view" src="https://user-images.githubusercontent.com/2132776/57255646-74114a80-7022-11e9-9e17-ea7eb172efa7.png">

We can also start to explore our environment we just created. As a note you'll only see this if you're authorized within a given organization and it's not shared as part of a public project, so if you're just following the public links you won't be able to see these. If you've created this in your own account, you'll be able to see the new environment we created. If you click in to it, you can see the deployment history for that environment, which lets us track all the deployment jobs.

<img width="1440" alt="Deployment history in environments" src="https://user-images.githubusercontent.com/2132776/57255393-9a82b600-7021-11e9-9f22-1615abcbe1d9.png">

Switching tabs, we can also see the commit history to have full traceability on this environment. For us, it's the whole list of commits in the repo up to this point.

<img width="1437" alt="Commit history" src="https://user-images.githubusercontent.com/2132776/57254599-ae2d1d00-701f-11e9-958e-6f6009c9c4a0.png">

And that's it for a quick look through a simple multi-stage deployment to App Service with Azure Pipelines. Issues or PRs? Feel free to open either!

## Coming soon
Adding a bit of complexity...

## Reference: getting started on App Service
1. [Create a free Azure account](https://azure.microsoft.com/en-us/free/) if you don't already have one
2. [Follow this quick guide](https://docs.microsoft.com/en-us/learn/modules/host-a-web-app-with-azure-app-service/3-exercise-create-a-web-app-in-the-azure-portal?pivots=node) on creating an App Service for a Node app

