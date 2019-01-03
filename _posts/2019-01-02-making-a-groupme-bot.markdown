---
layout: post
title:  "Making a GroupMe Chat Bot"
date:   2019-01-02 19:30:00 -0500
categories: development
tags: [webhooks, python, api, groupme, projects]
---

I belong to a lively Fantasy NBA league and was thinking recently it would be nice to use something like [Hubot](https://hubot.github.com/) to drop data about our league into our GroupMe chat. GroupMe has a pretty easy-to-use bot API, so I thought I’d build a simple chatbot myself. I'm going to walk through the basics of what I did to get this thing up and running in no time at all.

First thing to do is to create your bot-- I created a bot for the group using the [bot form](https://dev.groupme.com/bots/new). They have an API to create bots as well, if that’s something you are looking for.

![groupme_developers](https://user-images.githubusercontent.com/5407256/50621384-927c0380-0ed3-11e9-9973-097cedc3e97f.png)

Fill out the form and don’t worry about the callback URL for now. We’ll fill this out later on once we have one.

Once you create the bot, you’ll see it listed on the Bots page:

![bot list](https://user-images.githubusercontent.com/5407256/50621391-9576f400-0ed3-11e9-9311-0db61adbfec7.png)

Great, we have our bot registered with GroupMe. Let’s get to the fun part where we actually get to build our bot. 
The way the bot works is that any time a message is posted in our GroupMe channel, GroupMe will send a POST request to our yet-to-be-defined callback URL with some data. Here’s a sample of what it looks like:

```
{
  "attachments": [],
  "avatar_url": "https://i.groupme.com/123456789",
  "created_at": 1302623328,
  "group_id": "1234567890",
  "id": "1234567890",
  "name": "John",
  "sender_id": "12345",
  "sender_type": "user",
  "source_guid": "GUID",
  "system": false,
  "text": "Hello world ☃☃",
  "user_id": "1234567890"
}
```

So what we need to do is build a web service that handles this data and makes some decisions based on payload contents. I’m going to use Python and Flask to build this service because it’s what I know and I think it’s pretty simple to use, but you can use anything you want that can handle a POST request.

We’ll need two Python modules to help us out: Flask and Requests. Flask is a web development framework and will be used to handle the POST request GroupMe sends us, and requests will be used to POST a message back to GroupMe. Quick note: it’s a good idea to use a [virtual environment](https://docs.python-guide.org/dev/virtualenvs/) to install these. 

```
> pip install Flask

[...]
Successfully installed Flask-1.0.2 Jinja2-2.10 MarkupSafe-1.1.0 Werkzeug-0.14.1 click-7.0 itsdangerous-1.1.0
> pip install requests

[...]
Successfully installed certifi-2018.11.29 chardet-3.0.4 idna-2.8 requests-2.21.0 urllib3-1.24.1
```

Let’s start our Python script now. We’ll first build out receiving and handling the POST request from GroupMe, so let’s import a couple things from flask to help us out. We’ll need the Flask class to initialize our application and request to get data from the GroupMe call.

```
from flask import Flask, request
import requests

app = Flask(__name__)
```

Next, we define our route that will get called. If you’re new to Flask, our route is defined by the `app.route` decorator and we only want to accept POST requests at this route.

```
@app.route('/', methods=['POST'])
def main():
```

The first thing we do inside our method is get the JSON data we were passed from GroupMe using the Flask request object

```
message_payload = request.get_json()
```

Easy enough. Now we want to handle the data and do something with it. As a simple example, we’ll just say hello to the person who sent the message. We can get this by examining the payload and building a simple message to send back.

```
sender = message_payload[‘name’]
response = f‘Hello {sender}!’
```

So how do we actually send this message back to our GroupMe channel? All we need to do is send a POST request to the GroupMe endpoint for bots with the correct payload, which includes our bot’s ID and the message we want it to send. We’ll build a small helper method to assist with this.

We need to import requests to use in our helper method.

```
import requests
```

And then our helper method

```
def respond(message_text):
   payload = {
       'bot_id': '{Your bot ID}',
       'text': message_text
   }

   requests.post('https://api.groupme.com/v3/bots/post', data=payload)
```

We build our dictionary that contains the bot ID (which you can get from the GroupMe bot page where we first created the bot) and the message to send. Next, we call the post method on the request module with the GroupMe endpoint, and our payload. That’s all there is to the response. Now we can finish out our route. Let’s use our helper method we just defined to send the message back.

```
def main():
    message_payload = request.get_json()

    sender = message_payload['name']
    response = f'Hello {sender}!'
    respond(response)
```

One important note, every message that is posted in the channel will run through this bot, including messages that the bot posts. We need to filter these messages out, so let’s wrap our helper method in an IF statement to avoid an endless loop of the bot talking to itself.

```
if message_payload['name'] != 'mybot':
    sender = message_payload['name']
    response = f'Hello {sender}!'
    respond(response)
```

We’ll also need to properly respond to the request. Flask requires a message and status code in a tuple. We’ll return “ok” and 200.

```
return 'ok', 200
```

The last thing we want to do is run our flask app when this script gets called. We can do that by calling run on the Flask app. Here’s the whole script.

```
from flask import Flask, request
import requests


app = Flask(__name__)

@app.route('/', methods=['POST'])
def main():
   message_payload = request.get_json()

   if message_payload['name'] != 'mybot':
       sender = message_payload['name']
       response = f'Hello {sender}!'
       respond(response)

   return 'ok', 200


def respond(message_text):
   payload = {
       'bot_id': '{Your bot ID}',
       'text': message_text
   }

   requests.post('https://api.groupme.com/v3/bots/post', data=payload)


if __name__ == '__main__':
   app.run(port=3000)
```

What is this port=3000 at the end? This is the port my webhook relay is using and we are binding the web service to it. In order to test out our bot without deploying to a server, we need to use some sort of relay service for the webhook to the local instance of our bot. We can leverage something like [smee.io] (https://smee.io/) to make this work locally. Smee will forward the POST from GroupMe to our local instance of the bot. You can follow the directions on how to set it up when you start a new channel from the homepage, but it’s only a couple commands to get it up and running.

Run the following to install

```
> npm install --global smee-client
```

And then connect using the URL you got from starting a new channel at [smee.io](https://smee.io):

```
> smee -u https://smee.io/{your smee URL}
```

You should see something like

```
> smee -u https://smee.io/{your smee channel}

Forwarding https://smee.io/{your smee channel} to http://127.0.0.1:3000/
Connected https://smee.io/{your smee channel}
```

Let’s tell GroupMe about this URL. Go back to the bot configuration in GroupMe and paste your smee URL in the Callback URL field

![Callback URL](https://user-images.githubusercontent.com/5407256/50621388-9445c700-0ed3-11e9-8b3a-49f393aea076.png)

Now GroupMe knows where to send messages. Let’s test this out! Go to the channel you tied your bot to and send a message. You should see smee log the request in your terminal.

```
> smee -u https://smee.io/{your smee channel}

Forwarding https://smee.io/{your smee channel} to http://127.0.0.1:3000/
Connected https://smee.io/{your smee channel}
{ Error: connect ECONNREFUSED 127.0.0.1:3000
    at TCPConnectWrap.afterConnect [as oncomplete] (net.js:1121:14)
  errno: 'ECONNREFUSED',
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 3000,
  response: undefined }
```

It worked! Smee just logs an error because nothing is listening on the port right now. Let’s fix that and fire up our web service.

```
> python demo.py
 * Serving Flask app "demo" (lazy loading)
 * Environment: production
   WARNING: Do not use the development server in a production environment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:3000/ (Press CTRL+C to quit)
```

And let’s send another message to see what happens in the channel.

![Successful Test!](https://user-images.githubusercontent.com/5407256/50621394-9740b780-0ed3-11e9-993e-4ef49935035f.png)

Success! We have made our very simple chat bot! So what’s next? Ideally, this doesn’t just run on your machine forever. I ended up setting up a small server on DigitalOcean that runs my bot. I also expanded the simple bot to pull data from our fantasy league and post it to our channel when summoned by very specific commands. There’s really no limitation to what you can do here.