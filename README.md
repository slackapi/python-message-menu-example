Maximizing your Slack app's usefulness
---------------------------------------

When we think about how bots and humans interact within Slack, we often think of two types of interaction: **Notification** and **Conversation**.

The simplest form of Slack integration is to have your app post important information from another service into a slack channel. This works well for a variety of situations, such as package arrival notifications or an alert that one of your servers has caught on fire. Notifications are simple, timely, and informative. When your app is a part of a much more complex workflow, notifications are just a jumping point into another application or process.

<notification message screenshot>

As your processes and workflows become more complex, involving several applications across multiple windows, things can get ...complicated.

Message Buttons
----------------

Back in June, we released Message Buttons. Using these new interactive elements, developers were able to bring common workflow actions right into the bot's message. Users could not only see the notification message, but take action from inside Slack. This also eased a lot of the complexity with conversational bots. Rather than requiring a user to type a specific confirmation message, you were able to show the user a set of buttons to complete or cancel an action.

![Message Buttons demo GIF](https://cdn-images-1.medium.com/max/800/1*aYzTFMBlg8tGnKP7fv7oJg.gif)

While Message Buttons are a great way to increase the usability of your Slack application, there are some limitations. One limitation is the amount of real estate buttons take up. When you have a large set of options, you'll end up with a message that takes up the user's entire chat window or gets truncated.

See our [Message Buttons with Node.js](https://api.slack.com/tutorials/intro-to-message-buttons) tutorial for more information on how to add message buttons to your app.

Message Menus
-------------

Today we've released **[message menus](https://api.slack.com/docs/message-menus)** :tada:

Message menus are the newest interactive feature for Slack apps: clickable dropdown menus that you can add to [message attachments](https://api.slack.com/docs/message-attachments). They can have static options, or they can load dynamically.
You can build with five types of message menu today, each achieving a different flavor of use case: static menus, user menus, channel menus, conversation menus, and dynamic menus.

![Slack message menu example](https://cdn-images-1.medium.com/max/800/1*lLR-3KbUjwPF9l6jEbiEwQ.gif)

Read more about how apps are using message menus on [our blog](https://medium.com/slack-developer-blog/build-an-interactive-slack-app-with-message-menus-1fb2c6298308).

Adding message menus to your app
--------------------------------

In this tutorial, we'll focus on adding dynamic option menus to your app. Here's a summary of how dynamic interactive message menus work:

1. Post a message containing one or more ``attachments`` containing one or more interactive elements
2. Slack sends a request to your registered Options Load URL containing the context you need to generate relevant menu options. You simply respond to this request with a JSON array of options.
3. Users click a button or select an option from a menu
4. A request is sent to your registered Action URL containing all the context you need to understand: who clicked it, which option they clicked, which message ``callback_id`` was associated with the message, and the original message inciting the selection
5. You respond to your Action URL's invocation with a message to replace the original, and/or a new ephemeral message, and/or you utilize the invocation's ``response_url`` to update the original message out of band for a limited time.


**Let's build it!**

:bulb: This code is also available as a complete example application on [GitHub](https://github.com/slackapi/python-message-menu-example/blob/master/example.py).

While we're building this example in Python, we haven't forgotten about the Node fans.
We've put together a really neat example for [Node.js over here](https://github.com/slackapi/sample-message-menus-node) :tada:

For this example, we're going to need [Python](https://www.python.org/), the Python Slack client ([slackclient](https://github.com/slackapi/python-slackclient)) and a webserver ([Flask](http://flask.pocoo.org/)).

First you'll create the basic elements of the app: A Slack client and webserver

```
from flask import Flask, request, make_response, Response
from slackclient import SlackClient
import json

# Your app's Slack bot user token
SLACK_BOT_TOKEN = os.environ["SLACK_BOT_TOKEN"]

# Slack client for Web API requests
slack_client = SlackClient(SLACK_BOT_TOKEN)

# Flask webserver for incoming traffic from Slack
app = Flask(__name__)
```

Then we'll add the two endpoints Slack will POST requests to. The first endpoint
is where Slack will send a request for items to populate the menu options,
we'll call this one ``message_options``.

```
@app.route("/slack/message_options", methods=["POST"])
def message_options():
    # Parse the request payload
    form_json = json.loads(request.form["payload"])

    menu_options = {
        "options": [
            {
                "text": "Chess",
                "value": "chess"
            },
            {
                "text": "Global Thermonuclear War",
                "value": "war"
            }
        ]
    }

    return Response(json.dumps(menu_options), mimetype='application/json')
```

The second endpoint you'll need is where Slack will send data when a user makes
a selection. We'll call this one ``message_actions``.

```
@app.route("/slack/message_actions", methods=["POST"])
def message_actions():

    # Parse the request payload
    form_json = json.loads(request.form["payload"])

    # Check to see what the user's selection was and update the message
    selection = form_json["actions"][0]["selected_options"][0]["value"]

    if selection == "war":
        message_text = "The only winning move is not to play.\nHow about a nice game of chess?"
    else:
        message_text = ":horse:"

    response = slack_client.api_call(
      "chat.update",
      channel=form_json["channel"]["id"],
      ts=form_json["message_ts"],
      text=message_text,
      attachments=[]
    )

    return make_response("", 200)
```

Now that our endpoints are configured, we can build the message containing the menu and send it to a channel.


In order to show the menu, we'll have to build the message attachment which will contain it.

```
message_attachments = [
    {
        "fallback": "Upgrade your Slack client to use messages like these.",
        "color": "#3AA3E3",
        "attachment_type": "default",
        "callback_id": "menu_options_2319",
        "actions": [
            {
                "name": "games_list",
                "text": "Pick a game...",
                "type": "select",
                "data_source": "external"
            }
        ]
    }
]
```

Once the attachment JSON is ready, simply post a message to the channel, adding the attachment containing the menu.

```
slack_client.api_call(
  "chat.postMessage",
  channel="C09EM2073",
  text="Shall we play a game?",
  attachments=message_attachments
)
```

Take note of the ``"data_source": "external"`` attribute in the attachment JSON. This is how Slack knows to pull the menu options from the ``message_options`` endpoint we set up above.

```
slack_client.api_call(
  "chat.postMessage",
  channel="C09EM2073",
  text="Shall we play a game?",
  attachments=attachments_json
)
```


Support
--------

Need help? Join [Bot Developer Hangout](http://dev4slack.xoxco.com/) and talk to us in [#slack-api](https://dev4slack.slack.com/messages/slack-api/).


You can also create an Issue right here on [GitHub](https://github.com/slackapi/python-message-menu-example/issues).
