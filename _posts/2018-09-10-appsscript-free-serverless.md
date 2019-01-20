---
layout: post
title: 'Going Serverless for free with Google Apps Script + Google Sheets'
description: >
  Building APIs, Message Queues, and simple databases for free with Google Apps Script
tags: [google-apps-script, javascript, googlisms, powershell]
comments: true
excerpt_separator: <!--more-->
---

> "We don't take advantage of our position. We keep saying no to free stuff, as we can afford it now." Lukas Forchhammer

<!--more-->

```powershell
Invoke-RestMethod -Uri $googleAppsScriptUri -Method Post -Body (@{sender="Slack";text="Hello!";token=$token}|ConvertTo-Json)
```

* [Background](#background)
* [Goals](#goals)
* [Prerequisites](#prerequisites)
* [Creating the initial resources](#creating-the-initial-resources)
    * [Google Sheet](#google-sheet)
    * [Google Apps Script](#google-apps-script)
        * [Updating the Apps Script manifest](#updating-the-apps-script-manifest)
        * [Adding the GET and POST method handlers](#adding-the-get-and-post-method-handlers)
        * [Adding your config](#adding-your-config)
        * [Adding the event handlers](#adding-the-event-handlers)
        * [Adding the message handlers](#adding-the-message-handlers)
        * [Adding in Queue Management helper functions](#adding-in-queue-management-helper-functions)
* [Final steps](#final-steps)
    * [1. Add the Sheets API to the Apps Script project in the Developer's Console](#1-add-the-sheets-api-to-the-apps-script-project-in-the-developers-console)
    * [2. Authorize the Apps Script for your account](#2-authorize-the-apps-script-for-your-account)
    * [3. Deploy the project as a WebApp](#3-deploy-the-project-as-a-webapp)
    * [4. Update your notification webhooks with the Apps Script execution URL](#4-update-your-notification-webhooks-with-the-apps-script-execution-url)
* [Dequeueing messages with GET requests](#dequeueing-messages-with-get-requests)
* [Wrapping up](#wrapping-up)
{:toc}

## Background

Lately, I've been getting into creating Chat bots more, especially extending the excellent [PoshBot](https://github.com/poshbotio/PoshBot) bot framework for PowerShell created by the talented Brandon Olin ([@devblackops](https://twitter.com/devblackops)). With the initial implementation of PoshBot for Slack, it relied on Slack's Real Time Messaging API essentially broadcasting everything via WebSocket connection. This approach was ideal for PoshBot as it enabled you to open a WebSocket to Slack from your local network and leverage tools and commands to administer your local hosts and services without opening up your local resources externally. Unfortunately, not all extensible Chat clients offer real-time connections like Slack, so I embarked on a new project to see if I could make magic happen for the low-cost goodness of **$0** with Google Hangouts' replacement, [Google Chat](https://gsuite.google.com/products/chat/).

> ... but why Google Apps Script vs Google Cloud Functions/AWS Lambda/Azure Functions/etc?
{:.lead}

The biggest reason (for me) is cost. FaaS (Functions as a Service), while typically inexpensive for my personal use cases, still has the potential to incur cost at some point. Google Apps Script is free to use and has pretty [reasonable quotas](https://developers.google.com/apps-script/guides/services/quotas)). Combine that with the ability to hook into other Google Services and you can create something really useful for no cost whatsoever. In an environment where the cost of running FaaS can be a drop in the bucket, `$free` should still not be overlooked just because the current solution is easily affordable. After all, saving cents makes sense and those bits do add up!

## Goals

The main goal that I had going into this project outside of eliminating any additional costs was to create a Message Queue for Chat messages that a bot framework like PoshBot can poll for new messages to process and react accordingly. I wanted to also enable this same process to act as a centralized webhook endpoint to receive, parse and fan out CI/CD notifications. This is useful as some services like [CircleCI](https://circleci.com/) only support a limited number of notification options.

With those targets in mind, here are the key points I laid out:
* Google Sheets will act as the following components:
  * Message Queue for Chat messages where a bot framework will poll via GET requests
  * Dead Letter Queue for any POSTs received that were not validated or sent from an unknown sender (for inspection later)
  * Logging (I opted to leverage the [BetterLog library by Peter Herrmann](https://github.com/peterherrmann/BetterLog) for this)
* Google Apps Script acting as the API layer to the Sheets backend handling GET and POST request validation
  * Communicating with Google Apps Script & Sheets via API (vs using the Apps Script Deployment ID as the Chat endpoint) has the added bonus of keeping that Sheets backend editable only by you, so there won't be random stumbling by others within your domain that could interfere with the Apps Script framework being built out.

## Prerequisites

To get started, all we're going to need is a Google account. You can use a free Gmail account to accomplish everything here, so no worries if you aren't a G Suite customer. The only real difference between a free Gmail account and a paid G Suite account in regards to Apps Script usage would be [quotas](https://developers.google.com/apps-script/guides/services/quotas), but even the free Gmail account should be able to accomplish everything here without being rate limited.

That being said, you _will_ need a G Suite account if you are looking to integrate this with Google Chat as I am here, but it's not essential to find this post useful.

## Creating the initial resources

### Google Sheet

First thing you will want to do is create a [new Google Sheet](https://sheets.google.com). Once created, give it a useful/descriptive name like _Awesome Sheets Backend_.

### Google Apps Script

Once the Sheet is created, let's hop over to the Script Editor for the Sheet by navigating to `Tools > Script Editor` in the Sheets menu bar. This will open up the Script Editor a new browser tab with the default `Code.gs` file open containing a starter function wrapper:

```js
function myFunction() {
  
}
```

Once open, let's give the Apps Script project a useful name as well like _Awesome Apps Script API_.

#### Updating the Apps Script manifest

Apps Script comes with a manifest file, `appsscript.json`, which is hidden by default. We're going to want to update this to tell Apps Script what Google Services and other resources the Script will need. To show the manifest, navigate to `View > Show manifest file` from Apps Script's menu bar. You should see a new file appear named `appsscript.json` with the following default content:

```json
{
  "timeZone": "America/Mexico_City",
  "dependencies": {
  },
  "exceptionLogging": "STACKDRIVER"
}
```

We're going to want to update that so our Apps Script project contains the following:
1. The Sheets advanced service
2. The BetterLog library
3. The WebApp permissions to allow anonymous access and set execution as the user deploying the WebApp (necessary for the API setup)
4. The API execution access to `ANYONE`

Copy/paste the following in your `appsscript.json` to get it set up as needed, then save:
> You may update the `timeZone` to your preference as well if you'd like; [here is a list of all valid `timeZone`'s that Apps Script supports](https://gist.github.com/mhawksey/8673e904a03a91750c26c2754fe0977a).
> If you do not want to send logs to StackDriver, replace `STACKDRIVER` with `NONE` next to `exceptionLogging` below.

```json
{
  "timeZone": "America/Mexico_City",
  "dependencies": {
    "enabledAdvancedServices": [{
      "userSymbol": "Sheets",
      "serviceId": "sheets",
      "version": "v4"
    }],
    "libraries": [{
      "userSymbol": "BetterLog",
      "libraryId": "1DSyxam1ceq72bMHsE6aOVeOl94X78WCwiYPytKi7chlg4x5GqiNXSw0l",
      "version": "27"
    }]
  },
  "webapp": {
    "access": "ANYONE_ANONYMOUS",
    "executeAs": "USER_DEPLOYING"
  },
  "exceptionLogging": "STACKDRIVER",
  "executionApi": {
    "access": "ANYONE"
  }
}
```

#### Adding the GET and POST method handlers

Let's keep our primary code file clean and add the essentials:
1. Override the default `Logger` with `BetterLog`
2. Add the `doGet(e)` function to enable GET request handling. In this function, we'll...
  1. log that a new GET request was received
  2. get the config dictionary using a `getConfig()` function defined later
  3. return the JSON results using Apps Script's `ContentService.createTextOutput()` with set MimeType
3. Add the `doPost(e)` function to enable POST request handling. In this function, we'll...
  1. log that a new POST request was received
  2. get the config dictionary using `getConfig()`
  3. parse the sender out using a custom function `parseSender(e, config)`
  4. process the POST request using a custom function `processPost(e, sender, config)`
  5. return a blank JSON object `{}` so the sender knows the POST was successful

In Script Editor, switch back over to the `Code.gs` file and overwrite the default contents with the following, then save:

```js
Logger = BetterLog.useSpreadsheet(SpreadsheetApp.getActiveSpreadsheet().getId());

function doGet(e) {
    Logger.log('------------ NEW GET REQUEST ------------');
    var config = getConfig();
    return ContentService.createTextOutput(JSON.stringify(processGet(e, config))).setMimeType(ContentService.MimeType.JSON);
}

function doPost(e) {
    Logger.log('------------ NEW POST REQUEST ------------');
    var config = getConfig();
    var sender = parseSender(e, config);
    processPost(e, sender, config);
    return ContentService.createTextOutput('{}').setMimeType(ContentService.MimeType.JSON);
}
```

#### Adding your config

In Script Editor, create a new file named `Config.gs`. The sample config skeleton below contains the following configuration elements:
* **APIKey**: This is where you would paste your own API key. This can be any string you'd like. Any API calls made will need to have this as either a URL query parameter named `token`, i.e. `https://script.google.com........./exec?token=mySuperSecretApiKey`, or as a property within the POST body named `token`.
* **GChat**: This section is used when leveraging this process as an API endpoint for Google Chat. `addToQueue: true` indicates that this message will be added to the Sheets Queue for processing via Bot framework.
* **Unknown**: This section allows you to specify how validated messages from unknown senders are handled. This is especially useful when you'd like to add another service that supports Slack webhooks as a notification endpoint, as you should just be able to add the Apps Script URL in place of a Slack webhook where indicated. Slack formatted messages will be passed through as-is and the format will be parsed out and forwarded to Google Chat as well, if desired.
* **The rest** _(AppVeyor, CircleCI, etc)_: These are sender-specific configs that allow you to direct the flow of relaying messages specific to each source.


Paste the following contents in the new `Config.gs` file to build your config skeleton and fill in the relevant portions:

```js
function getConfig() {
  var CONF = {
    "APIKey": "paste your own custom API key here", // Example: 9cd356da-3b1b-4ab5-9569-2f8e421c02a6
    "GChat": {
      "addToQueue": true, // Since I want incoming Google Chat messages to be added to the Queue, set this to true
      "verificationToken": "paste the verification token from Google Chat API's config page here"
    },
    "Unknown": {
      // This is where we handle any validated POSTs from unknown senders. Useful to allow any Slack formatted messages to passthru
      "addToQueue": false, // Add this message to the queue?
      "includeUserCard": true, // For GChat, should we build a card object and format the sender's icon and username?
      "passThru": true, // Should we allow the validated unknown sender's message to passthru?
      "destinations": {
        "GChat": ["paste GChat webhook here"],
        "Slack": [
          {
            "webhook": "paste Slack webhook here"
          }
        ]
      }
    },
    "AppVeyor": {
      "addToQueue": false, // Add this message to the queue?
      "includeUserCard": true, // For GChat, should we build a card object and format the sender's icon and username?
      "destinations": {
        "GChat": ["paste GChat webhook here"],
        "Slack": [
          {
            "webhook": "paste Slack webhook here"
          }
        ]
      },
      "icon": "https://upload.wikimedia.org/wikipedia/commons/thumb/b/bc/Appveyor_logo.svg/2000px-Appveyor_logo.svg.png"
    },
    "CircleCI": {
      "addToQueue": false, // Add this message to the queue?
      "includeUserCard": true, // For GChat, should we build a card object and format the sender's icon and username?
      "destinations": {
        "GChat": ["paste GChat webhook here"],
        "Slack": [
          {
            "webhook": "paste Slack webhook here"
          }
        ]
      },
      "icon": "https://static.brandfolder.com/circleci/logo/circleci-primary-logo.png"
    },
    "GitHub": {
      "addToQueue": false, // Add this message to the queue?
      "includeUserCard": true, // For GChat, should we build a card object and format the sender's icon and username?
      "destinations": {
        "GChat": ["paste GChat webhook here"],
        "Slack": [
          {
            "webhook": "paste Slack webhook here"
          }
        ]
      },
      "icon": "https://assets-cdn.github.com/images/modules/logos_page/GitHub-Mark.png"
    },
    "TravisCI": {
      "addToQueue": false, // Add this message to the queue?
      "includeUserCard": true, // For GChat, should we build a card object and format the sender's icon and username?
      "destinations": {
        "GChat": ["paste GChat webhook here"],
        "Slack": [
          {
            "webhook": "paste Slack webhook here"
          }
        ]
      },
      "icon": "https://www.ocadotechnology.com/wp-content/uploads/2018/02/TravisCI-Mascot-1.png"
    },
    "VSTS": {
      "addToQueue": false, // Add this message to the queue?
      "includeUserCard": true, // For GChat, should we build a card object and format the sender's icon and username?
      "destinations": {
        "GChat": ["paste GChat webhook here"],
        "Slack": [
          {
            "webhook": "paste Slack webhook here"
          }
        ]
      },
      "icon": "https://a3bf67a2345da5d6ee8b-6d37b1ee447a16ff81e1420be19ec8c3.ssl.cf5.rackcdn.com/vsts/vsts.png"
    }
  };
  return CONF;
}
```

#### Adding the event handlers

Let's create another file in our Apps Script project named `EventHandler.gs`. The functions in this file will focus on processing the event data. Here are the functions and their purpose for reference:

* `parseSender(event, config)`: Used to figure out who the sender of the message is
* `validateEvent(e, config)`: Confirms whether the event contains the correct token before continuing
* `processPost(event, sender, config)`: The function called once POSTs are validated. This handles adding items to the queue or fanning out the relayed messages to appropriate recipients.
* `processGet(event, config)`: The function called once GETs are validated. This will be used to dequeue messages from the Sheets Queue.

Paste the following into `EventHandler.gs` then save:

```js
function parseSender(event, config) {
    var tracker = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Tracker");
    var dlq = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Dead Letters");
    var idRange = tracker.getRange(2, 1);
    var nextId = idRange.getValue() + 1;
    try {
        var postData = {};
        var sender = { "matched": false };
        if ('postData' in event) {
            if (event.postData.type === 'application/x-www-form-urlencoded' && 'payload' in event.parameter) {
                postData = JSON.parse(event.parameter.payload);
            }
            else {
                postData = JSON.parse(event.postData.contents);
            }
        }
        if ('token' in postData && postData.token === config.GChat.verificationToken) {
            sender = {
                "format": "raw",
                "matched": true,
                "sender": "GChat"
            };
        }
        else if ('build_url' in postData && (/^https:\/\/travis-ci.org\/.*/).test(postData.build_url)) {
            sender = {
                "format": "raw",
                "matched": true,
                "sender": "TravisCI"
            };
        }
        else if ('eventData' in postData && 'buildUrl' in postData.eventData && (/^https:\/\/ci.appveyor.com\/.*/).test(postData.eventData.buildUrl)) {
            sender = {
                "format": "raw",
                "matched": true,
                "sender": "AppVeyor"
            };
        }
        else if ('sender' in postData && 'url' in postData.sender && (/^https:\/\/api.github.com\/users.*/).test(postData.sender.url)) {
            sender = {
                "format": "raw",
                "matched": true,
                "sender": "GitHub"
            };
        }
        else if ('resource' in postData && (/^https:\/\/.*.visualstudio.com\/.*/).test(postData.resource.url) && config.VSTS.repos.includes(postData.eventData.repositoryName)) {
            sender = {
                "format": "raw",
                "matched": true,
                "sender": "VSTS"
            };
        }
        else if (('attachments' in postData && 'text' in postData.attachments[0]) || ('text' in postData && 'channel' in postData)) {
            if ('attachments' in postData && 'text' in postData.attachments[0] && (/(https:\/\/circleci.com\/.*|^Hello from CircleCI$)/).test(postData.attachments[0].text)) {
                sender = {
                    "format": "slack",
                    "matched": true,
                    "sender": "CircleCI"
                };
            }
            else if ('attachments' in postData && 'text' in postData.attachments[0] && (/https:\/\/ci.appveyor.com\/.*/).test(postData.attachments[0].text)) {
                sender = {
                    "format": "slack",
                    "matched": true,
                    "sender": "AppVeyor"
                };
            }
            else if (config.Unknown.passThru === true) {
                sender = {
                    "format": "slack",
                    "matched": true,
                    "sender": "Unknown"
                };
            }
        }
        return sender;
    }
    catch (e) {
        var err = (typeof e === 'string') ?
            new Error(e) :
            e;
        Logger.severe('%s: %s (line %s, file "%s"). Stack: "%s" . While processing %s.', err.name || '', err.message || '', err.lineNumber || '', err.fileName || '', err.stack || '', '');
        dlq.appendRow([Utilities.formatDate(new Date(), Session.getScriptTimeZone(), 'yyyy-MM-dd HH:mm:ss:SSS Z'), nextId, JSON.stringify(event), event.postData.contents, 'Unknown[Not Validated]']);
        throw err;
    }
}

/**
 * Validates the JSON payload
 * 
 * @param {Object}
 * e the event to validate 
 */
function validateEvent(e, config) {
    var validation = { "success": false };
    if (typeof e !== 'undefined') {
        if ('postData' in e) {
            var postData = {};
            if (e.postData.type === 'application/x-www-form-urlencoded' && 'payload' in e.parameter) {
                postData = JSON.parse(e.parameter.payload);
            }
            else {
                postData = JSON.parse(e.postData.contents);
            }
            var postToken = null;
            if ('token' in postData) {
                postToken = postData.token;
            }
            if (postToken !== null && postToken === config.GChat.verificationToken) {
                validation = {
                    "message": "POST Payload token validated!",
                    "success": true
                };
            }
            else if (e.parameter.token === config.APIKey || e.parameter.token === config.GChat.verificationToken || ('token' in postData && (postToken === config.APIKey || postToken === config.GChat.verificationToken))) {
                validation = {
                    "message": "POST token validated!",
                    "success": true
                };
            }
        }
        else if (e.parameter.token === config.APIKey || e.parameter.token === config.GChat.verificationToken) {
            validation = {
                "message": "GET Token validated!",
                "success": true
            };
        }
    }
    Logger.log('Event Validation: ' + JSON.stringify(validation));
    return validation;
}

/**
 * Validates the event.token and adds the event to the message queue
 *
 * @param {Object}
 * event the event object from the API call
 * 
 * @param {String} sender the sender of the event returned from parseSender(e)
 */
function processPost(event, sender, config) {
    var validation = validateEvent(event, config);
    var sendConf = config[sender.sender];
    var tracker = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Tracker");
    var dlq = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Dead Letters");
    var idRange = tracker.getRange(2, 1);
    var nextId = idRange.getValue() + 1;
    var err = null;
    var postData = {};
    try {
        if (event.postData.type === 'application/x-www-form-urlencoded' && 'payload' in event.parameter) {
            postData = JSON.parse(event.parameter.payload);
        }
        else {
            postData = JSON.parse(event.postData.contents);
        }
    }
    catch (e) {
        err = (typeof e === 'string') ?
            new Error(e) :
            e;
        Logger.severe('%s: %s (line %s, file "%s"). Stack: "%s" . While processing %s.', err.name || '', err.message || '', err.lineNumber || '', err.fileName || '', err.stack || '', '');
        dlq.appendRow([Utilities.formatDate(new Date(), Session.getScriptTimeZone(), 'yyyy-MM-dd HH:mm:ss:SSS Z'), nextId, JSON.stringify(event), JSON.stringify(postData), JSON.stringify(sender)]);
        throw err;
    }
    if (validation.success) {

        if (sendConf.addToQueue) {
            var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Queue");
            idRange.setValue(nextId);
            if (sender.matched) {
                Logger.log("Event validated! Adding event to Sheets MQ");
                sheet.appendRow([nextId, JSON.stringify(postData), "No", sender.sender]);
            }
            else {
                Logger.log("Sender not matched but event was validated! Adding full event to Dead Letters queue for inspection");
                dlq.appendRow([Utilities.formatDate(new Date(), Session.getScriptTimeZone(), 'yyyy-MM-dd HH:mm:ss:SSS Z'), nextId, JSON.stringify(event), JSON.stringify(postData), 'Unknown[Validated]']);
            }
        }
        else if (sender.matched) {
            if ('destinations' in sendConf) {
                Logger.log("Fanning out message from sender [" + sender.sender + "] to webhook recipients");
                var parsed = parseMessage(postData, sender, config);
                var i = 0;
                if ('Slack' in sendConf.destinations) {
                    for (i = 0; i < sendConf.destinations.Slack.length; i++) {
                        try {
                            if (sender.format === 'slack') {
                                sendSlackMsg(parsed.message, sendConf.destinations.Slack[i].webhook, parsed.username, parsed.iconUrl, sendConf.destinations.Slack[i].channel || parsed.channel, parsed.color, postData);
                                Logger.log("Slack message sent for sender [" + sender.sender + "] with original postData!");
                            }
                            else {
                                sendSlackMsg(parsed.message, sendConf.destinations.Slack[i].webhook, parsed.username, parsed.iconUrl, parsed.channel, parsed.color);
                                Logger.log("Slack message sent for sender [" + sender.sender + "] with parsed postData!");
                            }
                        }
                        catch (e) {
                            err = (typeof e === 'string') ?
                                new Error(e) :
                                e;
                            Logger.severe('%s: %s (line %s, file "%s"). Stack: "%s" . While processing %s.', err.name || '', err.message || '', err.lineNumber || '', err.fileName || '', err.stack || '', '');
                            dlq.appendRow([Utilities.formatDate(new Date(), Session.getScriptTimeZone(), 'yyyy-MM-dd HH:mm:ss:SSS Z'), nextId, JSON.stringify(event), JSON.stringify(parsed), JSON.stringify(sender)]);
                        }
                    }
                }
                if ('GChat' in sendConf.destinations) {
                    for (i = 0; i < sendConf.destinations.GChat.length; i++) {
                        try {
                            sendGChatMsg(parsed.message, sendConf.destinations.GChat[i], true, parsed.messageHtml, parsed.username, parsed.iconUrl);
                            Logger.log("Google Chat message sent for sender [" + sender.sender + "] with parsed postData!");
                        }
                        catch (e) {
                            err = (typeof e === 'string') ?
                                new Error(e) :
                                e;
                            Logger.severe('%s: %s (line %s, file "%s"). Stack: "%s" . While processing %s.', err.name || '', err.message || '', err.lineNumber || '', err.fileName || '', err.stack || '', '');
                            dlq.appendRow([Utilities.formatDate(new Date(), Session.getScriptTimeZone(), 'yyyy-MM-dd HH:mm:ss:SSS Z'), nextId, JSON.stringify(event), JSON.stringify(parsed), JSON.stringify(sender)]);
                        }
                    }
                }
            }
            else {
                Logger.log("Sender [" + sender.sender + "] does not have the destinations property set on the config! Skipping.");
            }
        }
    }
    else {
        err = new Error("POST request not validated! Adding to Dead Letters sheet for inspection");
        Logger.severe('%s: %s (line %s, file "%s"). Stack: "%s" . While processing %s.', err.name || '', err.message || '', err.lineNumber || '', err.fileName || '', err.stack || '', '');
        dlq.appendRow([Utilities.formatDate(new Date(), Session.getScriptTimeZone(), 'yyyy-MM-dd HH:mm:ss:SSS Z'), nextId, JSON.stringify(event), JSON.stringify(postData), 'Unknown[Not Validated]']);
    }
}

/**
 * Validates the event.token and adds the event to the message queue
 *
 * @param {Object}
 * event the event object from the API call
 * 
 * @param {String} sender the sender of the event returned from parseSender(e)
 */
function processGet(event, config) {
    var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Queue");
    var validation = validateEvent(event, config);
    var curMaxRows = sheet.getMaxRows();
    var eventsToReturn = [];
    var matchCount = 0;
    var eventValue = null;
    if (validation.success) {
        var sourceType = (typeof event.parameters.source === 'undefined') ? ['GChat'] :
            event.parameters.source;
        var maxRows = (typeof event.parameter.maxRows === 'undefined') ?
            1 :
            event.parameter.maxRows;
        Logger.log("GET request validated! Dequeueing up to [" + maxRows + "] unacked row(s) for source type(s) [" + sourceType.toString() + "]");
        if (curMaxRows > 1) {
            var rows = sheet.getRange(1, 1, curMaxRows, sheet.getMaxColumns()).getValues();
            for (var c = 0; c < curMaxRows; c++) {
                if (sourceType.indexOf(rows[c][3]) > -1) {
                    if (rows[c][2] === 'No') {
                        eventValue = (event.parameter.format == 'json') ?
                            JSON.parse(rows[c][1]) :
                            rows[c][1];
                        eventsToReturn.push({
                            "Acked": rows[c][2],
                            "Event": eventValue,
                            "Id": rows[c][0],
                            "Source": rows[c][3]
                        });
                    }
                    if (event.parameter.dequeue != 'no') {
                        sheet.deleteRow(c - matchCount + 1);
                    }
                    matchCount += 1;
                    if (matchCount >= maxRows) {
                        break;
                    }
                }
            }
        }
        Logger.log("Returning [" + eventsToReturn.length + "] events from the queue matching source type(s) [" + sourceType.toString() + "]!");
        if (eventsToReturn.length === 0) {
            return {};
        }
        else {
            return eventsToReturn;
        }
    }
    else {
        var err = new Error("GET request not validated! Adding to Dead Letters sheet for inspection");
        Logger.severe('%s: %s (line %s, file "%s"). Stack: "%s" . While processing %s.', err.name || '', err.message || '', err.lineNumber || '', err.fileName || '', err.stack || '', '');
        SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Dead Letters").appendRow([Utilities.formatDate(new Date(), Session.getScriptTimeZone(), 'yyyy-MM-dd HH:mm:ss:SSS Z'), 'N/A', JSON.stringify(event), '{}', 'GET[Not Validated]']);
        throw err;
    }
}
```

#### Adding the message handlers

We'll also need a way to send messages to Slack and Google Chat as well as parse out the received POSTs into the necessary format. Create a new file named `MessageHandler.gs` in your Apps Script project.

Functions included here and their purpose for reference:
* `sendSlackMsg(message, webhook, username, iconUrl, channel, color, existingPayload)`: This sends out a Slack message from Apps Script using a Slack webhook. If the event has an existing payload built for Slack already, we'll prefer that so messages are received the same way as originally sent.
* `sendGChatMsg(message, webhook, includeUserCard, messageHtml, username, iconUrl)`: This sends out a Google Chat message from Apps Script via Google Chat webhook. If you specify `true` for `includeUserCard` on the config for the sender, this will create a Chat Card object with the sender information and icon shown, then send the message text as an HTML formatted `textParagraph` widget on the Card.
* `parseMessage(postData, sender, config)`: This parses out the message details into an object with common formatting usable by both Slack and Google Chat.

Paste the following into `MessageHandler.gs` and save:

```js
function sendSlackMsg(message, webhook, username, iconUrl, channel, color, existingPayload) {
    var payload = {};
    if (typeof existingPayload !== 'undefined') {
        payload = existingPayload;
        if (username !== null && (payload.username === null || typeof payload.username === 'undefined')) {
            payload.username = username;
        }
        if (iconUrl !== null && (payload.icon_url === null || typeof payload.icon_url === 'undefined')) {
            payload.icon_url = iconUrl;
        }
        if (channel !== null && (payload.channel === null || typeof payload.channel === 'undefined')) {
            payload.channel = channel;
        }
        if (color !== null && 'attachments' in payload && (payload.attachments[0].color === null || typeof payload.attachments[0].color === 'undefined')) {
            payload.attachments[0].color = color;
        }
    } else {
        payload = {
            "attachments": [{
                "color": color || null,
                "fallback": message,
                "mrkdwn_in": ["text"],
                "text": message
            }],
            "channel": channel || null,
            "icon_url": iconUrl || null,
            "username": username || null
        };
    }
    var options = {
        'contentType': 'application/json',
        'method': 'post',
        'payload': JSON.stringify(payload)
    };
    return UrlFetchApp.fetch(webhook, options);
}

function sendGChatMsg(message, webhook, includeUserCard, messageHtml, username, iconUrl) {
    var payload = { "fallbackText": message };
    if (includeUserCard === true) {
        if (typeof username !== 'undefined') {
            payload.cards = [{ "header": { "title": username } }];
            if (typeof iconUrl !== 'undefined') {
                payload.cards[0].header.imageUrl = iconUrl;
                payload.cards[0].header.imageStyle = 'IMAGE';
            }
        }
        if (typeof messageHtml !== 'undefined' && messageHtml !== null) {
            if (!('cards' in payload)) {
                payload.cards = [{}];
            }
            payload.cards[0].sections = [{ "widgets": [{ "textParagraph": { "text": messageHtml } }] }];
        } else {
            payload.text = message;
        }
    } else {
        payload.text = message;
    }
    var options = {
        'contentType': 'application/json',
        'method': 'post',
        'payload': JSON.stringify(payload)
    };
    return UrlFetchApp.fetch(webhook, options);
}

function parseMessage(postData, sender, config) {
    try {
        var parsed = {
            "channel": null,
            "color": null,
            "iconUrl": null,
            "message": null,
            "messageHtml": null,
            "payload": null,
            "username": null
        };
        Logger.log('Parsing message from [' + sender.sender + '] with format [' + sender.format + ']');
        switch (sender.format) {
            case 'raw':
                switch (sender.sender) {
                    case 'AppVeyor':
                        parsed.username = 'AppVeyor CI (GAS)';
                        parsed.iconUrl = config.AppVeyor.icon || 'https://ci.appveyor.com/assets/images/appveyor-blue-144.png';
                        parsed.color = (postData.eventName.indexOf('success') > -1) ?
                            '#41aa58' :
                            '#ffff00';
                        parsed.message = "<" + postData.eventData.buildUrl + "|[" + postData.eventData.projectName + "] AppVeyor Build " + postData.eventData.buildVersion + " " + postData.eventData.status + ">\r\nCommit <" + postData.eventData.commitUrl + "|" + postData.eventData.commitId + "> by " + postData.eventData.commitAuthor + " on " + postData.eventData.commitDate + ": _" + postData.eventData.commitMessage + "_";
                        parsed.messageHtml = '<a href="' + postData.eventData.buildUrl + '">[' + postData.eventData.projectName + ']</br></br>AppVeyor Build ' + postData.eventData.buildVersion + ' ' + postData.eventData.status + '</a><br>Commit <a href="' + postData.eventData.commitUrl + '">' + postData.eventData.commitId + '</a> by ' + postData.eventData.commitAuthor + ' on ' + postData.eventData.commitDate + ': <i>' + postData.eventData.commitMessage + '</i>';
                        break;
                    case 'GitHub':
                        parsed.username = 'GitHub (GAS)';
                        parsed.iconUrl = config.GitHub.icon || 'https://assets-cdn.github.com/images/modules/logos_page/GitHub-Mark.png';
                        if ('pusher' in postData) {
                            parsed.message = postData.pusher.name + " has pushed to GitHub repo <" + postData.repository.html_url + "|" + postData.repository.full_name + ">\n<" + postData.compare + "|Compare>";
                            parsed.messageHtml = postData.pusher.name + ' has pushed to GitHub repo <a href="' + postData.repository.html_url + '">' + postData.repository.full_name + '</a><br><a href="' + postData.compare + '">Compare</a>';
                            parsed.color = '#1bcee2';
                        } else if ('description' in postData && (postData.description.indexOf('build') > -1 || postData.description.indexOf('running') > -1 || postData.description.indexOf('tests') > -1)) {
                            parsed.message = "GitHub Build Update: <" + postData.target_url + "|" + postData.description + "> for repo <" + postData.repository.html_url + "|" + postData.repository.full_name + ">\nContext: _" + postData.context + "_";
                            parsed.messageHtml = 'GitHub Build Update: <a href="' + postData.target_url + '">' + postData.description + '</a> for repo <a href="' + postData.repository.html_url + '">' + postData.repository.full_name + "</a><br>Context: <i>" + postData.context + "</i>";
                            parsed.color = (postData.description.indexOf('passed') > -1 || postData.description.indexOf('succeeded') > -1) ?
                                '#41aa58' :
                                '#ff8040';
                        } else {
                            parsed.message = "GitHub Repo Update: <" + postData.target_url + "|" + postData.description + "> for repo <" + postData.repository.html_url + "|" + postData.repository.full_name + ">\nContext: _" + postData.context + "_";
                            parsed.messageHtml = 'GitHub Repo Update: <a href="' + postData.target_url + '">' + postData.description + '</a> for repo <a href="' + postData.repository.html_url + '">' + postData.repository.full_name + "</a><br>Context: <i>" + postData.context + "</i>";
                            parsed.color = '#959595';
                        }
                        break;
                    case 'TravisCI':
                        parsed.username = 'TravisCI (GAS)';
                        parsed.iconUrl = config.TravisCI.icon || 'https://www.ocadotechnology.com/wp-content/uploads/2018/02/TravisCI-Mascot-1.png';
                        parsed.color = (postData.result_message === 'Passed') ?
                            '#41aa58' :
                            '#ffff00';
                        parsed.message = "[<" + postData.compare_url + "|" + postData.repository.name + ">] <" + postData.build_url + "|TravisCI Build " + postData.number + "> status: *" + postData.status_message + "*";
                        parsed.messageHtml = '[<a href="' + postData.compare_url + '">' + postData.repository.name + '</a>]<br><a href="' + postData.build_url + '">TravisCI Build ' + postData.number + "</a> status: <b>" + postData.status_message + "</b>";
                        break;
                    case 'VSTS':
                        break;
                    default:
                        break;
                }
                break;
            case 'slack':
                parsed.payload = postData;
                if (sender.sender === 'CircleCI') {
                    parsed.username = 'CircleCI (GAS)';
                    parsed.iconUrl = config.CircleCI.icon || 'https://static.brandfolder.com/circleci/logo/circleci-primary-logo.png';
                } else {
                    parsed.username = postData.username || null;
                    parsed.iconUrl = postData.icon_url || null;
                }
                if ('channel' in postData && postData.channel !== null && postData.channel.length > 0) {
                    parsed.channel = postData.channel;
                }
                if ('attachments' in postData && 'color' in postData.attachments[0] && postData.attachments[0].color !== null && postData.attachments[0].color.length > 0) {
                    parsed.color = postData.attachments[0].color;
                }
                if ('text' in postData && postData.text.length > 0) {
                    parsed.message = postData.text;
                } else if ('attachments' in postData && 'text' in postData.attachments[0] && postData.attachments[0].text !== null && postData.attachments[0].text.length > 0) {
                    parsed.message = postData.attachments[0].text;
                }
                break;
            default:
                break;
        }
        Logger.log('Message parsed: ' + JSON.stringify(parsed));
        return parsed;
    } catch (e) {
        var err = (typeof e === 'string') ?
            new Error(e) :
            e;
        Logger.severe('%s: %s (line %s, file "%s"). Stack: "%s" . While processing %s.', err.name || '', err.message || '', err.lineNumber || '', err.fileName || '', err.stack || '', '');
        throw err;
    }
}
```

#### Adding in Queue Management helper functions

On to the last file we'll need in our project, the `QueueMgmt.gs` file. This will contain one core function, `cleanupSheet()`, which cleans the acked rows from the Queue sheet, adds any Sheets it finds missing (i.e. Dead Letters and Tracker sheets), and sets some formatting on each Sheet to increase readability.

Paste the following on the new `QueueMgmt.gs` file and save:

```js
/**
 * Cleans up the Sheet and removes any Acked or empty rows
 */
function cleanupSheet() {
  Logger.log('------------ CLEANING UP SHEET ------------');
  // Get Sheet objects
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("Queue");
  var tracker = ss.getSheetByName("Tracker");
  var dlq = ss.getSheetByName("Dead Letters");
  // Create Sheets if missing
  if (!sheet) {
    Logger.log("Queue sheet not found! Creating...");
    ss.insertSheet("Queue", 0);
    sheet = ss.getSheetByName("Queue");
    sheet.deleteRows(2, (sheet.getMaxRows() - 1));
  }
  if (!tracker) {
    Logger.log("Tracker sheet not found! Creating...");
    ss.insertSheet("Tracker");
    tracker = ss.getSheetByName("Tracker");
    var trackerRange = tracker.getRange(1, 1, 2);
    trackerRange.setValues([["EventId"],["1"]]);
    tracker.deleteRows(3, (tracker.getMaxRows() - 2)).deleteColumns(2, (tracker.getMaxColumns() - 1));
  }
  if (!dlq) {
    Logger.log("Dead Letter Queue sheet not found! Creating...");
    ss.insertSheet("Dead Letters");
    dlq = ss.getSheetByName("Dead Letters");
    var dlqRange = dlq.getRange("A1:E1");
    dlqRange.setValues([["DateTime","Id","Event","PostData","Source"]]);
    dlq.deleteRows(2, (dlq.getMaxRows() - 1));
  }
  // Remove unneccessary columns and rows
  if (sheet.getMaxColumns() > 4) {
    sheet.deleteColumns(5, (sheet.getMaxColumns() - 4));
  }
  if (tracker.getMaxColumns() > 2) {
    tracker.deleteColumns(3, (tracker.getMaxColumns() - 2));
  }
  if (tracker.getMaxRows() > 2) {
    tracker.deleteRows(3, (tracker.getMaxRows() - 2));
  }
  if (dlq.getMaxColumns() > 5) {
    dlq.deleteColumns(6, (dlq.getMaxColumns() - 5));
  }
  // Remove the default Sheet if present
  var sheetOne = ss.getSheetByName("Sheet1");
  if (sheetOne) {
    Logger.log("Default sheet1 found! Deleting...");
    ss.deleteSheet(sheetOne);
  }
  // Fix the queue sheet's header row if broken
  if (sheet.getRange(1,2).getValue() != "Event") {
    Logger.log("Inserting new row 1 for Header");
    sheet.insertRowBefore(1);
    var values = [["Id", "Event", "Acked", "Source"]];
    var range = sheet.getRange("A1:D1");
    range.setValues(values);
    range.setHorizontalAlignment("center").setFontWeight("bold");
  }
  // Set sheet style
  dlq.getRange("A1:C1").setHorizontalAlignment("center").setFontWeight("bold");
  tracker.getRange("A1").setHorizontalAlignment("center").setFontWeight("bold");
  tracker.getRange("A2").setHorizontalAlignment("center");
  sheet.getRange("B:B").setWrap(true).setWrapStrategy(SpreadsheetApp.WrapStrategy.WRAP);
  dlq.getRange("C:C").setWrap(true).setWrapStrategy(SpreadsheetApp.WrapStrategy.WRAP);
  sheet.setTabColor("41f4d9");
  tracker.setTabColor("f4df41");
  dlq.setTabColor("ff0000");
  // Checked for acked rows to delete from the Queue and delete them if found
  var toDelete = [];
  var deleteCount = 0;
  var curMaxRows = sheet.getMaxRows();
  var rows = sheet.getRange(1, 1, curMaxRows, sheet.getMaxColumns()).getValues();
  for (var c = 0; c < curMaxRows; c++) {
    if (rows[c][2] == "") {
      toDelete.push(rows[c][0]);
    }
    else if (rows[c][2] == "Yes") {
      toDelete.push(rows[c][0]);
    }
  }
  if (toDelete.length > 0) {
    rows = sheet.getRange(1, 1, curMaxRows, sheet.getMaxColumns()).getValues();
    for (var i = 0; i < curMaxRows; i++) {
      if (rows[i][0] != "Id") {
        for (var d = 0; d < toDelete.length; d++) {
          if (rows[i][0] == toDelete[d]) {
            var rowToDelete = i + 1 - deleteCount;
            sheet.deleteRow(rowToDelete);
            deleteCount += 1;
            break;
          }
        }
      }
    }
  }
  Logger.log("Cleaned up " + deleteCount + " rows.");
}
```

## Final steps

Alright, there are just a few final steps before we can start leveraging this endpoint:
1. Add the Sheets API to the Apps Script project in the Developer's Console
2. Authorize the Apps Script for your account
3. Deploy the project as a WebApp
4. Update your notification webhooks with the Apps Script execution URL


### 1. Add the Sheets API to the Apps Script project in the Developer's Console

Enabling the Sheets API for this project is pretty quick:
1. In the Script Editor's menu bar, navigate to `Resources > Advanced Google services`
2. Click the blue link on the bottom of the pop-up window to go to the **Google API Console**. You will be taken to the Developer Console's Dashboard for the Apps Script Project.
3. Click the blue button on top to **Enable APIs and Services**, then search for `Google Sheets API`
4. Click the Google Sheets API from the search results
5. Click the blue button to **Enable** the API for this project

### 2. Authorize the Apps Script for your account

Once we have the Sheets API enabled for the project, we need to authorize Apps Script to run on our behalf:
1. Open the `QueueMgmt.gs` file in the Apps Script Editor
2. In the Script Editor's menu bar, navigate to `Run > Run function > cleanupSheet`
3. You will receive a pop-up stating **Authorization required**. Click `Review Permissions`, select the Google account you used to create the project, then click the blue **Allow** button.

### 3. Deploy the project as a WebApp

Time to deploy our new WebApp so we can start making GET and POST requests:
1. In the Script Editor's menu bar, navigate to `Publish > Deploy as web app`
2. Select `New` as the version
3. Click the blue **Update** button to deploy your WebApp

Once deployed, you'll be able to copy the current web app URL. Copy that now, as you'll need it next! The web app URL should look similar to the following:

`https://script.google.com/macros/s/oBs4c7sCfz_2kCjrCN3OvbKh36_ZmYXhIpToeidKt_sdazF7yxbwpj4/exec`

### 4. Update your notification webhooks with the Apps Script execution URL

Finally, let's start plugging in our new webhook endpoint so we can start leveraging it.

The first thing you'll want to do is append your token to the URL parameter. Using the example URL above, you will end up with something like this:

`https://script.google.com/macros/s/oBs4c7sCfz_2kCjrCN3OvbKh36_ZmYXhIpToeidKt_sdazF7yxbwpj4/exec?token=mySuperSecretApiKey`

Since Google Apps Script cannot parse message headers (😲), this enables the request to be validated for processing.
> If you are using the webhook as a Google Chat API endpoint and you have the Chat API token included in your config, then you can simply paste the URL as-is to the Hangouts Chat API configuration page.

**AppVeyor**:  
1. Open the AppVeyor project you would like to add notifications for, then go to the Notifications section in the project's settings
2. Add a new notification type and choose either Slack or Webhook
3. Paste the URL with token parameter in the Webhook field and save.

**CircleCI**:  
1. Open the CircleCI project you would like to add notifications for, then go to the Chat Notifications section in the project's settings
2. Choose Slack, then paste the URL with token parameter in the Webhook field and save.

**TravisCI**:  
**IMPORTANT** TravisCI requires you to set up your notifications directly on the `.travis.yml` file. If you have this file in public source control, please make sure to encrypt the value per TravisCI's instructions. [The Slack notification type has a good example of doing this](https://docs.travis-ci.com/user/notifications/#configuring-slack-notifications), but you'd run something similar to the following:

`travis encrypt "https://script.google.com/macros/s/oBs4c7sCfz_2kCjrCN3OvbKh36_ZmYXhIpToeidKt_sdazF7yxbwpj4/exec?token=mySuperSecretApiKey" --add notifications.webhooks.urls`

1. Open your `.travis.yml` file in your project
2. Add the webhook under `notifications.webhooks.urls`. It should look similar to this once done, if storing as an encrypted string:

```yaml
language: cpp
python:
- '3.4'
git:
  depth: 1000
os:
- linux
dist: trusty
script: echo Build Done
notifications:
  webhooks:
    urls:
      secure: Rj3QNQEkj1JlByH1YG9baSXyUrNZfPffAYYYTLVxNc9jJ5fTkexqRL28PFVfLqg5vzk7qDBzL2I5AS8eWpwaUMeq20YLCNK5PDre6mGtwMK9nvBQGH5vN0mKwPcmFindr6e81VmnbEPI+IO4KRBI7tW3PnKH4+ag9+MHcBhbsB/frMJoP9I0tgmw4GWUpvkyGNRoa4pUdIxbk7AsOqO//R/FX24NDrOgVvaqJCoHTD7zRfbW8IL5ul8nQq/gxh/3GjK2sQqcuPcxx6kzI8OaZwWS0xlF+nYider6PbjhUi+M5EUeY3aXlblZjXIpqrKk1EOXqncOLZg5h9s4dZWO3QKK7UEdK6qPVq+MkrishRYF6uVOMIvr9he50/g+UueUp2t5Al5k2jUGUGQsGgO5gSaBzWRJq3nMkYNvfQrD786/Bg/ZtHBgvrSj2NDSY65kp2pwbtufkLy3muvnBQbRIVf9aeywGjNrmvgOle+aZWjlwRvsav4u5Ont44jwkto81aYnE/7YYn4PekUd5Kc9SAH+vD2wfhejQfTU44IYbPt4jQUNCt0Fo4oyamUYEzn+0LqKlFHY4U++AwdruxmvPPY7vHzeSkOkrjYZpgJsnwdp9iq+jGm9cptvmSL2c3CnN2uX//pzS38zPCPRcbmPvnWlxkhLkA+OAqk1mFoTRk8=
```

**GitHub**:  
1. Open your repo on GitHub, then go to Settings
2. Click the Webhooks section on the left side
3. Click the **Add webhook** button on the top-right of the page
4. Paste the URL with token parameter in the Payload URL field
5. Leave `Content type` as `application/x-www-form-urlencoded` (Apps Script returns a 302 Redirect response when using JSON)
6. Choose the events you would like to trigger this webhook for.
7. Click the green **Add webhook** button to add the webhook and save.

GitHub will send a ping to the webhook to confirm it's operational


## Dequeueing messages with GET requests

Now that everything is set up for POSTing messages, here's a quick runthrough on dequeueing messages with GET requests.

GET requests leverage the following URL query parameters to get the unacked messages we need from the Sheet Queue:

```yml
parameter: token
description: The token to validate the request with
required: true
example: https://script.google.com/macros/s/oBs4c7sCfz_2kCjrCN3OvbKh36_ZmYXhIpToeidKt_sdazF7yxbwpj4/exec?token=mySuperSecretApiKey
```

```yml
parameter: dequeue
description: Set this to no to retrieve events from the queue without removing them
values: 
  - yes
  - no
default: yes
required: false
example: https://script.google.com/macros/s/oBs4c7sCfz_2kCjrCN3OvbKh36_ZmYXhIpToeidKt_sdazF7yxbwpj4/exec?token=mySuperSecretApiKey&dequeue=no
```

```yml
parameter: source
description: The source name you would like to retrieve messages for. You can set multiple values to retrieve multiple source types in one call, as seen in the example below.
values:
  - GChat
  - AppVeyor
  - CircleCI
  - GitHub
  - Slack
  - etc
default: GChat
required: false
example: https://script.google.com/macros/s/oBs4c7sCfz_2kCjrCN3OvbKh36_ZmYXhIpToeidKt_sdazF7yxbwpj4/exec?token=mySuperSecretApiKey&source=GChat&source=Slack
```

PowerShell example showing retrieval of Google Chat messages, with `dequeue` set to `no` to test message retrieval without removing from the queue:

```powershell
$uri = [Uri]"https://script.google.com/macros/s/oBs4c7sCfz_2kCjrCN3OvbKh36_ZmYXhIpToeidKt_sdazF7yxbwpj4/exec?token=mySuperSecretApiKey&dequeue=no"
$messages = Invoke-RestMethod -Uri $uri -Method Get -ContentType 'application/json'
# rest of code to handle message processing
```

## Wrapping up

OK, that was wordy! If you've made it this far, thank you for taking the time to read through the post!

If you have any questions, comments, etc, feel free to leave a comment below!

Please note that this is an ongoing project, so I may be updating the post content in case anything needs adjusting!



Cheers!

- Nate