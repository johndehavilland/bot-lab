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

2. Setup Slack as a platform for your bot - details here: [https://docs.microsoft.com/en-us/bot-framework/channel-connect-slack](https://docs.microsoft.com/en-us/bot-framework/channel-connect-slack)

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

## Create a Natural Language Understanding App
1. First go to [luis.ai](https://luis.ai) and create/login to your LUIS account. LUIS stands for Language Understanding Intelligent Service. You can find out more details [here](https://docs.microsoft.com/en-us/azure/cognitive-services/LUIS/Home)

2. Once logged in, you need to create an application. Give it a name and description and click create.

![Create a LUIS app](/images/create_luis.png)

3. Now you need to create an intent by clicking on *create an intent*

4. The first intent we will add is a *search* intent. Enter this and press save.

5. Now we need to define some utterances that will trigger this intent. These will be example phrases that a user may use that would expect to trigger this search intent. Examples could be
![Defined utterances](/images/defined_utterances.png)

6. When you have defined a few utterances, Save these and then navigate to *Entities*

7. Choose *Add a Custom Entity* and create a entity called *keywords*. This will basically be used to extract the keywords from a submitted user phrase.

8. Now switch back to Intents and choose the Search intent. Within the Search intent we now need to flag the entity for each entity. Do this by click on a word (or more) and choose the entity defined in step 7. 
![Adding entity](/images/defined_keywords.png)

9. When this is complete, choose Save and then navigate to *Train and Test*

10. Once complete, you can try it out and see how it performs. Type in a phrase and see if the LUIS app detects the right intent and also extracts the right part.

11. Finally, switch back over to publish and choose Publish to publish your app. Copy the endpoint for future use.

## Integrate NL into your bot.

1. We are now going to integrate this natural language into your bot. Add the following code to your bot just before the `server.post

```javascript
var LUIS_URL = "<luis_url>"
// Create a LUIS recognizer for our bot, to identify intents from
// user text.
const recognizer = new builder.LuisRecognizer(LUIS_URL);

// Create our bot to listen in on the chat connector.
global.bot = new builder.UniversalBot(connector, (session) => {
    session.beginDialog('sobot:search')
});

bot.recognizer(recognizer);

bot.use(builder.Middleware.sendTyping());

// Sends a nice greeting on a new message.
bot.on('conversationUpdate', (message) => {
    if (!message.membersAdded) {
        return;
    }

    message.membersAdded.forEach((identity) => {
        if (identity.id !== message.address.bot.id) {
            return;
        }

        bot.send(new builder.Message()
            .address(message.address)
            .text(`👋 Hello! I'm Stack Overflow's Resident Expert Bot 🤖 \
                and I'm here to help you find questions, answers, or to \
                just entertain you with a joke. Go ahead - ask me something!`
            ));
    });
});
```

3. Right after this, above the `server.post` add the following:

bot.dialog('search', [
    async (session, args) => {
        session.sendTyping();
        let userText = session.message.text.toLowerCase();
        searchQuery(session, { query: userText });
    }
]).triggerAction({
    matches: ['search']
});

bot.dialog('unknown', [
    async (session, args) => {
        session.sendTyping();
        session.endDialog("Did not understand. Try phrasing your ask a little differently");
    }
]).triggerAction({
    matches: ['None']
});

var searchQuery = async (session, args) => {
    return session.endDialog("You triggered a search for "+ args.query);
}

4. Replace the  `server.listen` with the following:

```javascript
server.listen(process.env.port || process.env.PORT || 3978, () => {
    console.log('%s listening to %s', server.name, server.url);
});
```


5. Run the app and try it in the emulator. You should see certain phrases trigger the search intent. Since we only have the Search intent and the None intent you will probably trigger the Search intent with each phrase.

## Integrate in Search functionality.

1. For this we will use the Bing Custom Search service. Navigate to [https://customsearch.ai/](https://customsearch.ai/) and sign in.

2. Choose New Custom Search and enter a name.

3. On the new app page that appears, enter stackoverflow.com as the website like so (important, do not enter http://www before it):
![Stack Overflow](/images/stackoverflow.png)

4. Click the endpoint icon next to the app name to navigate to the endpoint page:
![End point page](/images/stackoverflow_endpoint.png)

5. Take a copy of the primary key (press the reveal icon) and custom configuration id.

6. Add the following replacing the searchQuery - keep the server.post code intact below this:

```javascript
var searchQuery = async (session, args) => {
    let searchResults = await fetchSearchResults(args.query);
    
    // Process search results
    if (searchResults && searchResults.length > 0) {
        session.send("I found the following results from your question...");
        session.send(buildResultsMessageWithAttachments(session, searchResults));
        return session.endDialog("Feel free to ask me another question, or even ask for a joke!");
    } else {
        return session.endDialog('Sorry… couldnt find any results for your query! 🤐');
    }
}
```

var fetchSearchResults = async (query) => {
    var searchResults = [];

    await bingSearchClient(query, (err, res) => {
        if (err) {
            console.error('Error from callback:', err);
        } else if (res && res.webPages && res.webPages.value && res.webPages.value.length > 0) {
            for (var index = 0; index < res.webPages.value.length; index++) {
                var val = res.webPages.value[index];
                var result = {
                    title: val.name,
                    body_markdown: val.snippet,
                    link: val.url
                };
                searchResults.push(result);
            }
        }
    });

    return searchResults;
}

const buildResultsMessageWithAttachments = (session, resultsArray) => {
    const attachments = [];

    const message = new builder.Message(session);
    message.attachmentLayout(builder.AttachmentLayout.carousel);

    //Just to be safe, skype and teams have a card limit of 6/10
    let limit = (resultsArray.length > 6) ? 6 : resultsArray.length;

    for (let i = 0; i < limit; i++) {
        const result = resultsArray[i];

        const attachment = {
            contentType: "application/vnd.microsoft.card.adaptive",
            content: {
                type: "AdaptiveCard",
                body: [
                    {
                        "type": "ColumnSet",
                        "columns": [
                            {
                                "type": "Column",
                                "size": 2,
                                "items": [
                                    {
                                        "type": "TextBlock",
                                        "text": `${result.title}`,
                                        "weight": "bolder",
                                        "size": "large",
                                        "wrap": true,
                                    },
                                    {
                                        "type": "TextBlock",
                                        "text": `${result.body_markdown}`,
                                        "size": "normal",
                                        "horizontalAlignment": "left",
                                        "wrap": true,
                                        "maxLines": 5,
                                    }
                                ]
                            }
                        ]
                    }
                ],
                actions: [
                    {
                        "type": "Action.OpenUrl",
                        "title": "Find out more",
                        "url": `${result.link}`
                    }
                ]
            }
        }

        attachments.push(attachment);
    }

    message.attachments(attachments);
    return message;
};

var bingSearchClient = async (userSearchText, cb) => {

    var bingSearchConfig = "<config_code>";
    var bingSearchKey = "<primary_key>";
    var bingSearchCount = 6;
    var bingSearchMkt = "en-us";
    var bingSearchBaseUrl = "https://api.cognitive.microsoft.com/bingcustomsearch/v5.0/search";
    var bingSearchMaxSearchStringSize = 150;
    
    cb = cb || (() => {});

    const searchText = userSearchText.substring(0, bingSearchMaxSearchStringSize).trim();

    const url = bingSearchBaseUrl + "?"
                + `q=${encodeURIComponent(searchText)}`
                + `&customconfig=${bingSearchConfig}`
                + `&count=${bingSearchCount}`
                + `&mkt=${bingSearchMkt}`
                + "&offset=0&responseFilter=Webpages&safesearch=Strict";

    const options = {
        method: 'GET',
        uri: url,
        json: true,
        headers: {
            "Ocp-Apim-Subscription-Key": bingSearchKey
        }
    };

    await rp(options)
        .then((body) => {
            // POST succeeded
            return cb(null, body);
        })
        .catch((err) => {
            // POST failed
            return cb(err);
        });
}
```

7. Run this in the emulator and you should see it return cards for questions to Stack Overflow.

8. Now you can push this to your Azure repository and try it out in Slack.
