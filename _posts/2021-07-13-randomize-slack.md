---
layout:      posts
background:  shorterBackground
title:       "Randomize your Slack Avatar"
subtitle:    ""
date:        2021-07-13 12:00
author:      "Drew Garrett"
headerImg:   "/assets/images/posts/code.jpg"
description: ""
---

## Randomize Your Slack Avatar with Azure Functions

I enjoy having some fun with my Slack avatars which are almost never a photo of myself. With inspiration from [Phil Cryer's post]({% post_url 2020-09-28-randomize-twitter %}) last year, I decided to implement a random avatar using an [Azure Function](https://docs.microsoft.com/en-us/azure/azure-functions/) and images from [https://thispersondoesnotexist.com/](https://thispersondoesnotexist.com/). The images provided are all AI generated and not real people. In the samples below, you can see there are some malformations that stand out.

{: style="text-align: center" }
![Sample 1](/assets/images/posts/2021-07-13-randomize-slack/sample1-small.jpg)
![Sample 2](/assets/images/posts/2021-07-13-randomize-slack/sample2-small.jpg)
![Sample 3](/assets/images/posts/2021-07-13-randomize-slack/sample3-small.jpg)
![Sample 4](/assets/images/posts/2021-07-13-randomize-slack/sample4-small.jpg)
![Sample 5](/assets/images/posts/2021-07-13-randomize-slack/sample5-small.jpg)
![Sample 6](/assets/images/posts/2021-07-13-randomize-slack/sample6-small.jpg)

## Overall Requirements

In order to perform this work, we will need a Slack Workspace and account and an Azure Subscription. The app **must** be built using a Windows machine for the proof of concept to work. It is entirely possible to switch to [another OS supported by .NET 5](https://github.com/dotnet/core/blob/main/release-notes/5.0/5.0-supported-os.md), however the Azure Function app will need to be modified accordingly. You will also be required to create a new Slack App as a part of this. This permission may be restricted by your Slack Workspace administrators.

### Required Slack Permissions

Slack works in an OAuth 2.0 setup where permissions are defined as scopes. For this application we need [users.profile:write](https://api.slack.com/scopes/users.profile:write). This permission allows the application to update the user's profile information (i.e. name, email, etc.) as well as update or remove their avatar. We will utilize the [users.setPhoto](https://api.slack.com/methods/users.setPhoto) web request to do our work.

## Install and Setup the Application

### Get the Code

Our POC code is in the `single-token-poc` branch.

```bash
git clone https://github.com/ocelotconsulting/randomize-avatar.git
cd randomize-avatar
git checkout single-token-poc
```

### Slack Configuration

#### Install the Slack App

1. Browse to [https://api.slack.com/apps](https://api.slack.com/apps)
2. Click on "Create New App"
3. Choose "From an app manifest" to continue
4. Choose the appropriate workspace for this application
5. Click Next
6. Paste the contents of `slack-app/manifest.yml` into the `YAML` tab
7. Click Next
8. Review the setup information and click Confirm when you're satisfied

You may be prompted to "Install to Workspace". If so, click the button to do so and follow the prompts before continuing.

#### Retrieve the Slack User Access Token

After your app is created, it will appear in your apps list at [https://api.slack.com/apps](https://api.slack.com/apps). Open that app and browse to "OAuth & Permissions" in the menu. Copy the `User OAuth Token` presented at the top of the page. It must begin with `xoxp-` to be valid.

### Azure Infrastructure

Our [Azure Function App](https://docs.microsoft.com/en-us/azure/azure-functions/) only needs the function app itself, a storage account, and an app service plan. We will utilize the consumption plan to keep costs [extremely low](https://azure.microsoft.com/en-us/pricing/details/functions/).

#### Setup the Azure Infrastructure

You can utilize the portal to configure your own Azure Function App, however we have provided a template you can use. You will need to change the names of the resources however.

1. Create a Resource Group to use
2. Open the Resource Group
3. Click Create -> Custom Deployment
4. Click "Build your own template in editor"
5. Paste the contents of `azure/functionapp.json` into the text box and click Save
6. Click "Edit parameters"
7. Paste the contents of `azure/functionapp.parameters.json` into the text box and click Save
   * This step can be skipped if you wish to edit the parameters directly in the portal instead
8. Change the values of the Name, Location, Hosting Plan Name, Storage Account Name, or another parameter as requried
9. Click "Review + create"
10. Confirm the parameters look correct and review the terms provided
11. Click Create

#### Configure the Function App

In order to access the Slack API, we need to provide the `User OAuth Token` gathered earlier. In a normal setup, this would be stored securely in an `Azure Key Vault`, however this is a proof of concept and the app will be deleted soon so we will simply use the `Function App Application Settings` to save it.

1. Open the Function App in the Azure Portal
2. Click Configuration on the left menu
3. On the Application Settings tab, click "New application setting"
4. Name: `SlackToken`
5. Value: `<Token Copied Earlier>`
6. Click OK
7. Click Save
8. Click Continue

### Deploy the Function App

There are many ways to deploy an Azure Function App. We are going to perform a manual deployment in this example.

1. Build and Publish (prepares for publish) the function app locally: `dotnet publish`
2. Browse to `bin/Debug/5.0/publish`
3. Zip the contents of the folder
4. Open the function App in the Azure Portal
5. Click "Advanced Tools" in the menu
6. Click Go to open a new tab
7. Go to Tools -> Zip Push Deploy
8. Drag and drop, or click Upload, the ZIP file you created earlier
9. Confirm that `slack-avatar.dll` and other files are in the top-level folder, if they're not you may need to recreate the ZIP file
10. Go back to the Function App in the Azure Portal
11. Click Functions in the menu
12. Confirm `UpdateAvatar` is now in the list, you may need to wait 1 minute and click Refresh
13. Open `UpdateAvatar`
14. Click "Code + Test" in the menu
15. Click "Test/Run" to trigger a test
16. Verify the avatar has updated in Slack

In testing, the "Test/Run" button was not always accessible. You can utilize the [manual run method](https://docs.microsoft.com/en-us/azure/azure-functions/functions-manually-run-non-http). Here is a sample cURL request:

```powershell
curl --request POST -H 'x-functions-key:[KEY]' -H "Content-Type:application/json" --data "{}" 'http://[FUNCAPP_NAME].azurewebsites.net/admin/functions/UpdateAvatar'
```

## Slack Example

How does it look in Slack? Here's an example:

{: style="text-align: center" }
![Slack Sample](/assets/images/posts/2021-07-13-randomize-slack/slack-sample.png)

## What's Next?

If you wish to modify how frequently the function app updates your avatar, edit `UpdateAvatar.cs` and modify the `TimerTrigger()` [attribute](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer?tabs=csharp#configuration) to different values. This is a [NCRONTAB expression](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer?tabs=csharp#ncrontab-expressions) which includes seconds. Only use the NCRONTAB format for now as you cannot use a `TimeSpan` with a consumption app service plan.

Example CRONs:

```plain
0 0 * * * *     - Hourly at 0 minutes
0 0,30 * * * *  - Every 30 minutes
0 0 0 * * *     - Daily at midnight (Timezone based on Function App, default is UTC)
0 0 * */2 * *   - Hourly but only on even numbered days
```

It would be great to expand this function to accept multiple users in the same workspace with registration via the Slack UI instead of the API website. That may be for a future blog post.

## Reminder to Cleanup Resources

As always, these are resources that cost money in Azure. Be sure to clean up the resources to prevent further billing once you're done using them. You should also delete the Slack App, or at a minimum revoke the `User OAuth Token` for security purposes.

## Part Two

Please check out [part two of this series]({% post_url 2021-07-22-randomize-slack-2 %}): expanding to multiple users and multiple tenants!
