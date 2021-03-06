# alexa-bridge

A bridge between Alexa Skills and the Microsoft Bot SDK

# Overview

This is a simple restify-based server that acts as a bridge between an Alexa Skill and a Microsoft Bot Framework bot. Utterances originating at an Alexa endpoint e.g. an Echo device are received by this bridge, translated and forwarded to the Microsoft Bot. Replies coming back from the Bot are then returned to the originating endpoint. Alexa will read out the text in the `Text` field of the Bot reply.

# HowTo

## Set up the Microsoft Bot

Head on over to the [Microsoft Bot Framework](https://dev.botframework.com/) and create yourself a new bot (if you didn't already have one). Set up a DirectLine channel. This is how the bridge will talk to the Bot.

## Set up ngrok (for local dev/testing)

Get yourself [ngrok](https://ngrok.com). To test both the bridge and the bot on your local machine you'll need two ngrok instances running.

```
ngrok http 8080 <- The bridge
ngrok http 3979 <- The Bot
```
If the bot is already published you can use the live end point from the bot configured through the DirectLine Channel above.

## Set up an Alexa Skill

First you'll need an [Alexa skill](https://developer.amazon.com). Set this up to be a Custom skill with whatever Invocation Name you prefer. 

### Interaction Model
Next, configure the interaction model exactly like this:
```json
{
  "intents": [
    {
      "slots": [
        {
          "name": "phrase",
          "type": "phrase"
        }
      ],
      "intent": "GetUserIntent"
    }
  ]
}
```

Next add a single custom slot type named 'phrase'. Value must be more than one word I used "this is a long sentence with multiple words". This is important as a single word here only returns one word to the end point.

Finally provide a single sample utterance:

```
GetUserIntent {phrase}
```

This ensures that everything the user says will be passed straight through to the Microsot Bot. We're not going to be using any of the intent or entity recognition features of Alexa instead we are using Alexa as a glorified microphone and speaker that performs the voice to text component for us.

### Configuration

Take the public ngrok endpoint for the 8080 port and use it to configure an HTTP endpoint for the skill in the Alexa Skill configuration.

```
ngrok http -bind-tls=true -host-header=rewrite 8080
```

Use the https protocol and add `/messages`. Final url should look something like this: `https://<id>.ngrok.io/messages`.

Later when you publish this solution to an Azure App Service you can change this to something like
`https://<your app name>.azurewebsites.net/messages`.

### SSL Certificate

For SSL certificate use:
My development endpoint is a sub-domain of a domain that has a wildcard certificate from a certificate authority 

### Test

To test the solution type a phrase into the Service Simulator - Enter Utterance text field - this should invoke your application below as long as the alexa-bridge is running.

### Configure Bot

Take the public endpoint for the 3979 (or whatever you choose) port and use as the messaging endpoint in the configration for the Microsoft bot.

## Start the alexa-bridge

Now we can start the bridge, but let's do a quick bit of configuration first:

### Configuring the alexa-bridge

Create a file called in the solution called .env
It has five configuration settings:

* `botId` - The Bot identity in the conversation. This won't actually be seen anywhere at present.
* `directLineSecret` - The secret created when setting up the DirectLine channel for the Bot.
* `promptPhrase="Can I help you with anything else?"` - This is the phrase used to prompt the user to continue the conversation
* `msglocale="en-US"` - The locale for the message language
* `APPINSIGHTS_INSTRUMENTATIONKEY` - The instrumentation key for appinsignts - comment out the appinsights code if you aren't using it. [Check out the quickstart](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-nodejs-quick-start)

### Starting the alexa-bridge

If you are using VS Code open the folder where you cloned this repo and navigate to the View-ntegrated Terminal and run 
```
npm install --save
```
Next navigate to the server.js file and Debug - Start Debugging (F5)

Alternatively run

```
@localhost:~/src/alexa-bridge$ node ./server.js
Starting...
restify listening at http://[::]:8080
```
At this point you should be able to use the test page on the Alexa Skill configuration to send utterances to the bridge. You'll get no replies unless you are connected to a published bot but ngrok should show you connections are being made.

### Start the Bot

How you do this depends on whether you're in Node or C#, but I'm trusting you can figure this out.

### Publishing this project to an Azure App Service

Once you are satisfied with the bridge you will need to publish it to Azure.

[Publish using Azure CLI](https://docs.microsoft.com/en-us/azure/app-service/app-service-web-get-started-nodejs)
Alternatively you can do this using the Azure portal. One thing I did find is that you may need to use the B1 Basic or above for the app service plan and host it in the same location as the Alexa skill and the bot to reduce latency. Free or shared app services don't support always on which means there is a long startup time for the bot/bridge which can lead to timeouts as Alexa will only wait 8 seconds for a response before timing out.

[Watch](https://azure.microsoft.com/en-us/resources/videos/create-a-nodejs-site-deploy-from-github/) or [Read](https://docs.microsoft.com/en-us/bot-framework/deploy-bot-github) more on deploying node.js websites to azure.

The key thing is that once the app is deployed to the app service you need to go to application settings and load the values from the .env file into the app settings panel.

You may also need to regenerate the npm packages on the server using [Kudu](https://github.com/projectkudu/kudu/) navigate to https://*****.scm.azurewebsites.net (where ***** is the name of your Web App) and running npm install against the Debug console / CMD / site / wwwroot directory. 