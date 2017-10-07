# Intelligent Bot Lab

This lab walks through building out an intelligent bot on the Azure platform.

## Create the initial bot endpoint

1. Clone this repo.

2. Navigate to the app folder and run `npm install`

3. Start the app locally by running `node app.js`

3. Download and install [bot emulator](https://github.com/Microsoft/BotFramework-Emulator/releases/tag/v3.5.31)
	
4. Run the emulator and connect to the endpoint (http://localhost:3978/api/messages). You can leave the app id and password blank.

5. Make sure you can send and receive messages from your bot in the emulator.

## Create a slack channel for the bot
1. Go to  [https://dev.botframework.com/bots/new](https://dev.botframework.com/bots/new) and create a bot application by filling out the details. Leave the message endpoint blank for now.

2. Setup Slack as a platform for your bot - details here: https://docs.microsoft.com/en-us/bot-framework/channel-connect-slack

## Deploy the Bot to Azure

1. Open up the [Azure portal](https://portal.azure.com) and login 

2. Open the cloud shell and make sure it is in BASH mode.

3. Run the following commands, replacing the placeholders with your own values

```
rgname=<name of resource group>
myplanname=<name of app plan>
bothandle=<name of bot>

az group create --location eastus --name $rgname
az appservice plan create --name $myplanname --resource-group $rgname --sku FREE
az webapp create --name $bothandle  --resource-group $rgname --plan $myplanname
az webapp deployment user set --user-name <username> --password <password>
az webapp deployment source config-local-git --name $bothandle --resource-group $rgname --query url --output tsv
```	

4. Copy the url that is displayed and locally navigate to the bot app directory and run the following commands:

```
cd <MyGitRepo>
git remote add azure <URLResultFromLastStep>
git push azure master
```

5. Once that has completed, in the portal go to the web app, choose Application Settings and add the following two settings. Use the values from when you created your bot.

```
MICROSOFT_APP_ID
MICROSOFT_APP_PASSWORD
```	

6. Navigate to your bot application in the [bot portal](https://dev.botframework.com)	
7. Edit your bot and Update the message endpoint to point to your app as follows:
		https://<appurl>/api/messages
	
8. Try it out - you should be able to interact with the bot in Slack

## Adding Natural Language Understanding
		
	
	
