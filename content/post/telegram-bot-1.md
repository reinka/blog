+++ author = "Andrei Pöhlmann"
title = "How-to: (Dockerized) Telegram bot that queries FTX for crypto prices"
date = "2021-10-07"
description = "Telegram bot that runs in a Docker container and queries FTX for crypto prices"
tags = [
"telegram",
"Python",
"tutorial"
]
categories = [
"misc"
]

thumbnail= "images/telegram-bot.png"
+++

This posts presents a simple Python 3 implementation of a Telegram bot that queries the FTX API for crypto currency prices.
It will present only a minimal implementation, ignoring best practices and the like, just to get you started. So you might
not wanna push this code into production.

This bot will allow you to fetch current crypto prices via the following commands in a chat:

```bash
/cprice btc

crypto_bot:
BTC-PERP price: 54026.0
Last 01h: -0.52%
Last 24h: -1.33%
```

As can be also seen in the thumbnail of this post above.

# Prerequisites

1. [FTX](https://ftx.com/) account for an API key & secret
2. [Telegram](https://telegram.org/) account
3. Optional: [Docker](https://www.docker.com/). 

To create an FTX API key & secret, please follow [these instructions](https://help.ftx.com/hc/en-us/articles/360028807171-API-docs).

To create a Telegram bot, please follow the following instructions: [BotFather](https://core.telegram.org/bots#6-botfather).

If you have no prior Docker exprience, you might wanna ignore the Docker parts because this post won't give an introduction to it. 
It is not necessary to have Docker. You can create below scripts, install their dependencies and run them locally on your laptop.

# How will we do it?

We will use the Python library [`python-telegram-bot`](https://github.com/python-telegram-bot/python-telegram-bot) which 
provides a Python interface for the [Telegram Bot API](https://core.telegram.org/bots/api). You can simply install it via

```python
pip install python-telegram-bot
```
For our bot, we will use as a reference the offical [echobot.py](https://github.com/python-telegram-bot/python-telegram-bot/blob/master/examples/echobot.py)
example.

Additionally, we will use an FTX Python client for their REST API, provided by an official FTX example: [ftexchange/rest/client.py](https://github.com/ftexchange/ftx/blob/master/rest/client.py) 

In a final step, we will pack all our dependencies, scripts etc. into a Dockerfile to build a portable container of our bot. 
Our final directory structure will look like the following:

```bash
apoehlmann:~/workspace/telegram-bot$ tree
.
├── Dockerfile
├── bot.py
└── ftx_client.py

0 directories, 3 files
```

# Show me the code already
TLDR:

```Python
#!/usr/bin/env python
# pylint: disable=C0116,W0613
# This program is dedicated to the public domain under the CC0 license.

"""
Simple Bot to reply to Telegram messages.
First, a few handler functions are defined. Then, those functions are passed to
the Dispatcher and registered at their respective places.
Then, the bot is started and runs until we press Ctrl-C on the command line.
Usage:
/cprice symbol
Press Ctrl-C on the command line or send a signal to the process to stop the
bot.
"""
import logging

from telegram import Update, ForceReply
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackContext
from ftx_client import FtxClient


# Enable logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO
)

logger = logging.getLogger(__name__)

class TelegramBot:
    def __init__(self, api_key, crypto_client):
        self.crypto_client = crypto_client
        # Create the Updater and pass it your bot's token.
        self.updater = Updater(api_key)
        # Get the dispatcher to register handlers
        self.dispatcher = self.updater.dispatcher

        # on different commands - answer in Telegram
        self.dispatcher.add_handler(CommandHandler("cprice", self.cprice))

        unknown_handler = MessageHandler(Filters.command, self.unknown)
        self.dispatcher.add_handler(unknown_handler)

    def start_bot(self):
        # Start the Bot
        self.updater.start_polling()

        # Run the bot until you press Ctrl-C or the process receives SIGINT,
        # SIGTERM or SIGABRT. This should be used most of the time, since
        # start_polling() is non-blocking and will stop the bot gracefully.
        self.updater.idle()

    def cprice(self, update: Update, context: CallbackContext) -> None:
        # context.args is a list that contains all space separated strings
        # following the command, e.g. /cprice xrp btc -> ["xrp", "btc"]
        coins = context.args
        # get markets data via the crypto client
        self.markets = [coin for coin in self.crypto_client._get(f"markets")]
        text = ''
        for coin in coins:
            coin_upper = coin.upper()
            found = False

            for entry in self.markets:
                if entry['name'].startswith(coin_upper):
                    found = True
                    text += f'{entry["name"]} price: {entry["last"]}\n' \
                        f'Last 01h: {100*entry["change1h"]:.2f}%\n' \
                        f'Last 24h: {100*entry["change24h"]:.2f}%\n\n'
                    break
            if not found:
                text += f'Sorry, no crypto symbol starting with {coin.upper()} has been found.\n\n'
        context.bot.send_message(chat_id=update.effective_chat.id, text=text)

    def unknown(self, update, context):
        context.bot.send_message(chat_id=update.effective_chat.id, text="Sorry, I didn't understand that command.")

def main() -> None:
    """Start the bot."""
    ftx_api_key = 'your_ftx_api_key_from_your_ftx_profile'
    ftx_secret = 'your_ftx_api_secret_from_your_ftx_profile'
    ftx_client = FtxClient(api_key=ftx_api_key, api_secret=ftx_secret)
    
    bot = TelegramBot("your_telegram_bot_token_from_botfather", ftx_client)
    bot.start_bot()

if __name__ == '__main__':
    main()
```
# Step by step walkthrough

Note: For a more thoroughful walkthrough of the API classes used below, check the offical python-telegram-bot tutorial
[Your first bot](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Extensions-%E2%80%93-Your-first-Bot).

First of all, just copy the [ftexchange/rest/client.py](https://github.com/ftexchange/ftx/blob/master/rest/client.py) to
a file called `ftx-client.py`. Note, that you must also install the following dependencies:

```bash
pip install ciso8601 requests
```

Next, create a new Python file called `bot.py`. This file will contain the code of our bot which we will describe next.

### Init our `TelegramBot` class

Inspired by the official [echobot.py](https://github.com/python-telegram-bot/python-telegram-bot/blob/master/examples/echobot.py),
our bot class will be instantiated with a Telegram bot token and a crypto client (we use [dependency-injection](https://en.wikipedia.org/wiki/Dependency_injection) for the client): 

(don't worry about all the imports they 
will make more sense when you read on)

```python
#!/usr/bin/env python
# pylint: disable=C0116,W0613
# This program is dedicated to the public domain under the CC0 license.

"""
Simple Bot to reply to Telegram messages.
First, a few handler functions are defined. Then, those functions are passed to
the Dispatcher and registered at their respective places.
Then, the bot is started and runs until we press Ctrl-C on the command line.
Usage:
/cprice symbol
Press Ctrl-C on the command line or send a signal to the process to stop the
bot.
"""

import logging

from telegram import Update, ForceReply
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackContext
from ftx_client import FtxClient  # copy of https://github.com/ftexchange/ftx/blob/master/rest/client.py


# Enable logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO
)

logger = logging.getLogger(__name__)

class TelegramBot:
    def __init__(self, api_key, crypto_client):
        # Save a reference to the client: it will be used to 
        # query the exchange for crypto prices.
        self.crypto_client = crypto_client
```
### Updater & Dispatcher: continuously fetch and dispatch new updates
For our bot to work, we need a way for to continuously fetch new Telegram updates. There's a 
[`telegram.ext.Updater`](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.updater.html#telegram.ext.Updater)
class that does exactly this. It takes as an input the bot's API token. It even does a little bit more: it also 
creates a so-called [`Dispatcher`](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.dispatcher.html#telegram.ext.Dispatcher)
which dispatches all kinds of updates to its registered handlers.

```python
class TelegramBot:
    def __init__(self, api_key, crypto_client):
        # Save a reference to the client: it will be used to 
        # query the exchange for crypto prices.
        self.crypto_client = crypto_client

        # Create the Updater and pass it your bot's token.
        self.updater = Updater(api_key)
        # Get the dispatcher to register handlers
        self.dispatcher = self.updater.dispatcher
```

### (Command)Handlers: handle the dispatched updates

A handler is a class that handles a specific incoming update: our goal will be to handle incoming commands of the kind

```bash
/cprice symbol
```
where `symbol` corresponds to a crypto symbol, e.g. `BTC, ETH, FTT`, etc. I use `cprice` as an abbreviation for `crypto price` here. 
Feel free to call your command however you like.

We can achieve this handling by a so called [`CommandHandler`](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.commandhandler.html).
A `CommandHandler` takes as input the name of the command (in our case `"cprice"`) and the so-called callback function it
should call when receiving this command. We will simply call our callback function the same way, i.e. `def cprice(...)`. 
(Note below is also a handler that gracefully handles unknown commands by a user; you can skip that part, the bot will just
not react in that case)

```python
class TelegramBot:
    def __init__(self, api_key, crypto_client):
        # Save a reference to the client: it will be used to 
        # query the exchange for crypto prices.
        self.crypto_client = crypto_client

        # Create the Updater and pass it your bot's token.
        self.updater = Updater(api_key)
        # Get the dispatcher to register handlers
        self.dispatcher = self.updater.dispatcher

        # on the following command - answer in Telegram
        self.dispatcher.add_handler(CommandHandler("cprice", self.cprice))

        # if a user typed an unkown command, handle it here
        unknown_handler = MessageHandler(Filters.command, self.unknown)
        self.dispatcher.add_handler(unknown_handler)
```

### Callback function: called by the handler

This callback function will basically contain all the logic for parsing the crypto symbol and querying the crypto exchange,
in our example FTX, via the crypto client.

We will use a simple `GET /markets` request using the FTX's REST API. We can have a look [at its official documentation](https://docs.ftx.com/#get-markets)
to see what the returned data will look like, e.g.:

```json
{
  "success": true,
  "result": [
    {
      "name": "BTC-0628",
      "baseCurrency": null,
      "quoteCurrency": null,
      "quoteVolume24h": 28914.76,
      "change1h": 0.012,
      "change24h": 0.0299,
      "changeBod": 0.0156,
      "highLeverageFeeExempt": false,
      "minProvideSize": 0.001,
      "type": "future",
      "underlying": "BTC",
      "enabled": true,
      "ask": 3949.25,
      "bid": 3949,
      "last": 10579.52,
      "postOnly": false,
      "price": 10579.52,
      "priceIncrement": 0.25,
      "sizeIncrement": 0.0001,
      "restricted": false,
      "volumeUsd24h": 28914.76
    }
  ]
}
```
So the result will be a list of dicts, where each dict corresponds to a crypto symbol and its data. Our bot will iterate
through this list, do a simple check if the `"name"` starts with the `symbol` provided by the user, e.g. `/cprice btc`, and if so
display its `"last"` price, and the last `"change1h"` and `"change24h"` percentage change.

Note, the following code is pretty dumb, i.e. for each crypto symbol provided by the `context.args` (that is space separated 
strings in following the command e.g. `/cprice btc eth`) it iterates through the whole cyrpto data all over again. Needless to 
say it's highly inefficient. But it's trivial to understand which is the purpose of this post. 

```Python
    def cprice(self, update: Update, context: CallbackContext) -> None:
        coins = context.args
        # get markets data
        self.markets = [coin for coin in self.crypto_client._get(f"markets")]
        text = ''
        for coin in coins:
            coin_upper = coin.upper()
            found = False

            for entry in self.markets:
                if entry['name'].startswith(coin_upper):
                    found = True
                    text += f'{entry["name"]} price: {entry["last"]}\n' \
                        f'Last 01h: {100*entry["change1h"]:.2f}%\n' \
                        f'Last 24h: {100*entry["change24h"]:.2f}%\n\n'
                    break
            if not found:
                text += f'Sorry, no crypto symbol starting with {coin.upper()} has been found.\n\n'
        context.bot.send_message(chat_id=update.effective_chat.id, text=text)
```

And here's our (optional) `unknown` callback function:

```Python
def unknown(self, update, context):
        context.bot.send_message(chat_id=update.effective_chat.id, text="Sorry, I didn't understand that command.")
```

### Start the bot

One thing is missing: we need a class method that starts the bot:

```Python
class TelegramBot:
    
    ...
    
    def start_bot(self):
        # Start the Bot
        self.updater.start_polling()

        # Run the bot until you press Ctrl-C or the process receives SIGINT,
        # SIGTERM or SIGABRT. This should be used most of the time, since
        # start_polling() is non-blocking and will stop the bot gracefully.
        self.updater.idle()
```

Together with a main function, we can finalize our script:

```Python
def main() -> None:
    """Start the bot."""
    ftx_api_key = 'your_ftx_api_key_from_your_ftx_profile'
    ftx_secret = 'your_ftx_api_secret_from_your_ftx_profile'
    ftx_client = FtxClient(api_key=ftx_api_key, api_secret=ftx_secret)
    
    bot = TelegramBot("your_telegram_bot_token_from_botfather", ftx_client)
    bot.start_bot()

if __name__ == '__main__':
    main()
```

# Wait, what about Docker?

We can use a simple `Dockerfile` that creates a container with all dependencies:

```Dockerfile 
FROM ubuntu:20.04

RUN apt update && apt install -y python3-pip 
RUN pip install python-telegram-bot ciso8601 requests
RUN ln -nsf /usr/bin/python3 /usr/bin/python

RUN mkdir /bot
COPY bot.py /bot/bot.py
COPY ftx_client.py /bot/ftx_client.py

WORKDIR /bot
```

### Build it and run it

I'd like to repeat that this is not an example of how to run the bot in production. We're violating best practices such
as avoiding running an application as `root` inside Docker. So be careful where you deploy this container:

```Python 
 docker build -t telegram-bot . && docker run -it telegram-bot python bot.py
```

That's it. You can now open telegram and talk to your bot:

![Telegram bot example](/images/telegram-bot.png)