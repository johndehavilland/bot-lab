# Intelligent Bot Lab

This lab walks through building out an intelligent bot on the Azure platform. It has been adapted from the [Stack Overflow Bot Sample](https://github.com/Microsoft/BotFramework-Samples/tree/master/StackOverflow-Bot).

## Create the initial bot endpoint

1. Clone this repo.

2. Navigate to the app folder and run `npm install`

3. Start the app locally by running `node app.js`

3. Download and install [bot emulator](https://github.com/Microsoft/BotFramework-Emulator/releases/tag/v3.5.31)
	
4. Run the emulator and connect to the endpoint (http://localhost:3978/api/messages). You can leave the app id and password blank.

5. Make sure you can send and receive messages from your bot in the emulator.

## Create a Natural Language Understanding App
1. First go to [luis.ai](http://luis.ai) and create/login to your LUIS account. LUIS stands for Language Understanding Intelligent Service. You can find out more details [here](https://docs.microsoft.com/en-us/azure/cognitive-services/LUIS/Home)

2. Once logged in, you need to create an application. Give it a name and description and click create.

![Create a LUIS app](/images/create_luis.PNG)

3. Now you need to create an intent by clicking on *create an intent*

4. The first intent we will add is a *search* intent. Enter this and press save.

5. Now we need to define some utterances that will trigger this intent. These will be example phrases that a user may use that would expect to trigger this search intent. Examples could be
![Defined utterances](/images/defined_utterances.PNG)

6. When you have defined a few utterances, Save these and then navigate to *Entities*

7. Choose *Add a Custom Entity* and create a entity called *keywords*. This will basically be used to extract the keywords from a submitted user phrase.

8. Now switch back to Intents and choose the Search intent. Within the Search intent we now need to flag the entity for each utterance. Do this by click on a word (or more) and choose the entity defined in step 7. 
![Adding entity](/images/define_keywords.PNG)

9. When this is complete, choose Save and then navigate to *Train and Test*

10. Once complete, you can try it out and see how it performs. Type in a phrase and see if the LUIS app detects the right intent and also extracts the right part.

11. Finally, switch back over to publish and choose Publish to publish your app. Copy the endpoint for future use. **Note** under API Key, there should be a starter key automatically created for you.

## Integrate natural language processing into your bot.

1. We are now going to integrate this natural language into your bot. Add the following code to your bot just before the `server.post`. Note that the code below uses async/await which is available in Node 7.6 and above.

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

3. Delete:
```javascript
// Receive messages from the user and respond by echoing each message back (prefixed with 'You said:')
var bot = new builder.UniversalBot(connector, function (session) {
    session.send("You said: %s.", session.message.text);
});
```

4. Below the `server.post` add the following:

```javascript
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
```

5. Run the app and try it in the emulator. You should see certain phrases trigger the search intent. Since we only have the Search intent and the None intent you will probably trigger the Search intent with each phrase.

## Integrate in Search Functionality.

1. For this we will use the Bing Custom Search service. Navigate to [https://customsearch.ai/](https://customsearch.ai/) and sign in.

2. Choose New Custom Search and enter a name.

3. On the new app page that appears, enter stackoverflow.com as the website like so (important, do not enter http://www before it):
![Stack Overflow](/images/stackoverflow.PNG)

4. Click the endpoint icon next to the app name to navigate to the endpoint page:
![End point page](/images/stackoverflow_endpoint.PNG)

5. Take a copy of the primary key (press the reveal icon) and custom configuration id.

6. Add the following replacing the searchQuery (replace the placeholders with your values from customsearch):

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

```

7. Run this in the emulator and you should see it return cards for questions to Stack Overflow.


## Add Another Intent
1. The above LUIS model has just 2 intents - Search and None. Let's add in another intent to give your app a more human touch. A joke intent.

2. Go back to your LUIS app ([https://luis.ai](https://luis.ai)).

3. Under Intents choose *Add New Intent* and a new intent called *joke*

4. The goal of this intent is to allow the user to ask your bot for a joke. Train this intent with some ways a user may ask for a joke:

![Examples of utterances for joke intent](/images/joke_intent.PNG)

5. Save this and navigate to *Train and Test*. Train the model and test it out and make sure it responds accordingly.

6. Navigate to *Publish* and click publish to update the existing published model.

7. In your app, we now need to react when we see a joke intent. To do this add in the following code - feel free to modify to choose from a collection of jokes at random:

```javascript
bot.dialog('joke', [
    async (session, args) => {
        var joke = "A SQL query goes into a bar, walks up to two tables and asks, 'Can I join you?'";
        session.send(joke);
    }
]).triggerAction({
    matches: ['joke']
});
```

8. Try it out in the emulator, you should see if you ask for a joke it will now trigger this action instead of searching Stack Overflow.

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
cd bot-lab/app
git init
git add .
git commit -m "initial commit"
git remote add azure <URLResultFromLastStep>
git push azure master
```

5. Once that has completed, in the portal go to the web app, choose Application Settings and add the following two settings. Use the values from when you created your bot.

```
MICROSOFT_APP_ID
MICROSOFT_APP_PASSWORD
```	

6. Navigate to your bot application in the [bot portal](https://dev.botframework.com)	
7. Edit your bot and Update the message endpoint to point to your app as follows: `https://<appurl>/api/messages`
	
8. Try it out - you should be able to interact with the bot in Slack