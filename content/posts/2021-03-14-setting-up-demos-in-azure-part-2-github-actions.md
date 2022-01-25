---
author: João Antunes
date: 2021-03-14 16:00:00+01:00
layout: post
title: "Setting up demos in Azure - Part 2: GitHub Actions"
summary: "Building on the work from the previous post, in this one we make use of GitHub Actions to setup workflows for easy setup and tear down of a demo environment in Azure."
images:
- '/images/2021/03/14/setting-up-demos-in-azure-part-2-github-actions.png'
categories:
- azure
- dotnet
tags:
- 'infrastructure as code'
- 'arm templates'
- 'github actions'
slug: setting-up-demos-in-azure-part-2-github-actions
---

## Intro

[In the previous post](https://blog.codingmilitia.com/2021/03/07/setting-up-demos-in-azure-part-1-arm-templates/). we prepared an [ARM template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview) so we could setup (and tear down) a demo environment with a couple of commands.

It's already much better than doing it manually through the portal, but we can make our life even easier!

The current required steps to get the demo ready are:

- Run command to create Azure resource group
- Run command to create resources using the ARM template
- Deploy the demo application

In this post, we'll turn these three steps into one, by using GitHub actions to create a workflow that does it all.

I won't go into great specifics of GitHub Actions, as it was [already discussed it in a previous post](https://blog.codingmilitia.com/2020/12/22/getting-started-with-github-actions/), so the focus is on using it to deploy things to Azure.

## Deployment workflow

Let's get right into the main topic of the post, the deployment workflow.

In the `.github/workflows` folder, we create a file for this workflow, in this case I called it `deploy-all-the-things.yml`.

At the top added the name and the ways the workflow will be triggered.

```yaml
name: Deploy all the things
on: [workflow_dispatch]
```

I don't want the workflow to be triggered every time there's a push, merge or related kinds of triggers, using instead the `workflow_dispatch` trigger, which will allow us to start the workflow with a click of a button in GitHub's UI.

After this, added some variables to use when defining the various steps.

```yaml
env:
  RESOURCE_GROUP_NAME: 'SettingUpDemosInAzure'
  RESOURCE_GROUP_LOCATION: 'westeurope'
  DOTNET_VERSION: '5.0.*'
```

You'll probably recognize the resource group related values, as we used them in the previous post.

Then, we start defining the jobs and their steps.

```yaml
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@master
```

The job will run on Ubuntu latest, but this is a bit irrelevant, what's important is it runs somewhere that can run the Azure CLI, as well as build and deploy .NET applications

The first step is to checkout the repository, so we have access to the application and the ARM template.

Now we're getting into the parts specific to deploy Azure stuff, by logging in, using the [Azure Login GitHub Action](https://github.com/marketplace/actions/azure-login).

```yaml
- name: Login to Azure
  uses: azure/login@v1
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}
```

We'll see later where those credentials come from, for now just know that GitHub stores them encrypted and passes them into the workflow.

After logging in, we can finally start interacting with Azure, starting with creating the resource group, using the [Azure CLI GitHub Action](https://github.com/marketplace/actions/azure-cli-action).

```yaml
- name: Create resource group
  uses: Azure/CLI@v1
  with:
    inlineScript: |
      #!/bin/bash
      if $(az group exists --name ${{ env.RESOURCE_GROUP_NAME }}) ; then
        echo "Azure resource group already exists, skipping creation..."
      else
        az group create --name ${{ env.RESOURCE_GROUP_NAME }} --location ${{ env.RESOURCE_GROUP_LOCATION }}
        echo "Azure resource group created"
      fi
```

In the script (copied from the [ARM template deployment action docs](https://github.com/marketplace/actions/deploy-azure-resource-manager-arm-template)), we check if the resource group exists, if it does we skip creation, otherwise we create it.

After we know the resource group is in place, we can deploy the ARM template, by using the [available GitHub Action](https://github.com/marketplace/actions/deploy-azure-resource-manager-arm-template).

```yaml
- name: Deploy ARM template
  id: deploy-arm
  uses: azure/arm-deploy@v1
  with:
    resourceGroupName: ${{ env.RESOURCE_GROUP_NAME }}
    template: ./infrastructure/all-the-things.json
    parameters: administratorLoginPassword=${{ secrets.SQL_ADMIN_PASSWORD }}
```

Not much to say about it. We pass in the target resource group, the template file and the required parameters, which in this case is just the SQL administrator password, provided through GitHub secrets.

We also added an id to the step, so we can reference it from other steps. That's because later, when we want to deploy the demo application, we'll need the name of the created App Service resource, so I added that as an output of the ARM template, which will be made accessible by this step.

Before we can deploy the demo application, we need to build and publish it, so we get into familiar .NET territory here.

```yaml
- name: Setup .NET
  uses: actions/setup-dotnet@v1
  with:
    dotnet-version: ${{ env.DOTNET_VERSION }} 

- name: dotnet build and publish
  run: |
    dotnet restore ./src/SettingUpDemosInAzure.Web/
    dotnet build ./src/SettingUpDemosInAzure.Web/ -c Release
    dotnet publish ./src/SettingUpDemosInAzure.Web/ -c Release -o './binaries'
```

We've seen this [in a previous post](https://blog.codingmilitia.com/2020/12/22/getting-started-with-github-actions/), in fact, a more complex version of this, as we had more steps for tests, multiple operating systems and whatnot, which isn't very relevant for this scenario. Just setting up .NET, building and publishing the application so it can be deployed in the next step.

```yaml
- name: Deploy application
  uses: azure/webapps-deploy@v2
  with: 
    app-name: ${{ steps.deploy-arm.outputs.appName }}
    package: './binaries'
```

This final step uses the [Azure WebApp GitHub Action](https://github.com/marketplace/actions/azure-webapp) to deploy the produced binaries to App Service. Notice that we're using the output of the `deploy-arm` step to get the value for the  `app-name` parameter.

## Tear down workflow

Just as we can create a workflow to set things up, we can also create one to tear them down.

Tearing things down is much simpler, as it basically requires a single step: deleting the resource group.

```yaml
name: Bring it all down
on: [workflow_dispatch]

env:
  RESOURCE_GROUP_NAME: 'SettingUpDemosInAzure'

jobs:
  bring-it-all-down:
    runs-on: ubuntu-latest
    steps:
    - name: Login to Azure
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Delete resource group
      uses: Azure/CLI@v1
      with:
        inlineScript: |
          #!/bin/bash
          if $(az group exists --name ${{ env.RESOURCE_GROUP_NAME }}) ; then
            echo "Deleting Azure resource group..."
            az group delete --name ${{ env.RESOURCE_GROUP_NAME }} -y
            echo "Azure resource group deleted"
          else            
            echo "Azure resource group doesn't exist, skipping deletion"
          fi
```

As we can see, defined it to be triggered manually as well. After that, all we need is to login to Azure and then use the CLI to delete the group if it exists.

## Credentials and secret management

Ok, so in the previous sections, we spotted the usage of secrets in there, so let's look into it.

Starting with the most important one, the Azure credentials.

Using the Azure CLI (or other means), we can create a [service principal to authenticate with Active Directory](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals).

```bash
az ad sp create-for-rbac --name "A_SERVICE_PRINCIPAL_NAME" --sdk-auth --role contributor --scopes /subscriptions/A_SUBSCIPTION_ID
```

The `role` parameter is an important one, as it specifies what kind of permissions the service principal will have. In this case, setting `contributor` means it has "full access to manage all resources, but does not allow you to assign roles in Azure RBAC, manage assignments in Azure Blueprints, or share image galleries" ([from the docs](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)).

Running this command will return a JSON object like the following:

```json
{
  "clientId": "A_CLIENT_ID",
  "clientSecret": "A_CLIENT_SECRET",
  "subscriptionId": "A_SUBSCRIPTION_ID",
  "tenantId": "A_TENANT_ID",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```

This is what our workflow needs to login (the `AZURE_CREDENTIALS` we saw earlier), so we need to store it in GitHub secrets.

We head to the repository settings, then on the left menu, we have a "Secrets" entry.

{{< embedded-image "/images/2021/03/14/01-github-actions-secrets.png" "GitHub Actions secrets" >}}

After we click it we can see the existing secrets (we can already see the ones used in the workflows). We also have a button at the top to create a new secret.

Clicking to create a new secret takes us to the following page:

{{< embedded-image "/images/2021/03/14/02-github-actions-add-new-secret.png" "GitHub Actions add new secret" >}}

In the secret name field, we type the name we want to use in the workflow. In the value field, we put the actual secret. For the Azure credentials, we put the whole JSON object we got from the service principal creation command.

The other secret we require is the SQL administrator password, so we can follow the same steps, typing the desired password in the value form field.

## Deploying things

With everything in place, we can finally deploy (and tear down) things.

Click on the "Actions" entry in the repository top menu, select the workflow to run, click "Run workflow", select the branch and again "Run workflow" on the modal that popped up.

{{< embedded-image "/images/2021/03/14/03-github-actions-trigger-workflow-manually.png" "GitHub Actions trigger workflow manually" >}}

It'll take a bit, but eventually everything will be up and running.

{{< embedded-image "/images/2021/03/14/04-github-actions-workflow-result.png" "GitHub Actions workflow result" >}}

And at this point we can make a couple of requests to our sample application.

{{< embedded-image "/images/2021/03/14/05-test-the-api-with-a-post.png" "Test the API with a POST" >}}

{{< embedded-image "/images/2021/03/14/06-test-the-api-with-a-get.png" "Test the API with a GET" >}}

## Outro

Like in the previous post, I'm pretty sure there's tons more stuff to learn on this subject. I'm also sure there are a lot of best practices I'm throwing out the window, but for quick and dirty demo setup, I'm pretty happy with the results!

Throughout this post, we looked at:

- Setting up a workflow to create a resource group, deploy an ARM template and a web application
- Setting up a workflow to easily tear down the whole demo environment
- Credentials and secret management on GitHub
- Triggering the workflows

Links in the post:

- [What are ARM templates?](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/)
- [GitHub Action - Azure Login](https://github.com/marketplace/actions/azure-login)
- [GitHub Action - Azure CLI](https://github.com/marketplace/actions/azure-cli-action)
- [GitHub Action - Deploy Azure Resource Manager (ARM) Template](https://github.com/marketplace/actions/deploy-azure-resource-manager-arm-template)
- [GitHub Action - Azure WebApp](https://github.com/marketplace/actions/azure-webapp)
- [Azure built-in roles](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Setting up demos in Azure - Part 1: ARM templates](https://blog.codingmilitia.com/2021/03/07/setting-up-demos-in-azure-part-1-arm-templates/)
- [Getting started with GitHub Actions](https://blog.codingmilitia.com/2020/12/22/getting-started-with-github-actions/)

The source code for this post is in the [SettingUpDemosInAzure](https://github.com/joaofbantunes/SettingUpDemosInAzure) repository.

Thanks for stopping by, cyaz!
