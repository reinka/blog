+++ author = "Andrei Pöhlmann"
title = "Telegram crypto bot: adding inline mode"
date = "2021-10-10"
description = "Learn how to add the inline mode feature to your telegram bot"
tags = [
"telegram",
"Python",
"tutorial"
]
categories = [
"misc"
]

thumbnail = "images/telegram-bot/inline-mode.jpg"
+++

In this post we add the inline mode feature to our Telegram crypto bot which 
allows interacting with the bot by typing its username in the text input field in any
chat:

![Inline mode](/images/telegram-bot/inline-price.gif)

For more info, see the official documentation:
[https://core.telegram.org/bots/inline](https://core.telegram.org/bots/inline)

# Prerequisites 

We must first enable inline mode by talking to `@BotFather` 
using `/setinline`. Note that it might take a while until your Bot registers 
as an inline bot on your client. Restarting your Telegram App might speed up the process.
If it doesn't you just need to be a little bit patient. 

# Code

```Python
# new imports additional to the ones from our last post
from telegram import InlineQueryResultArticle, InputTextMessageContent
from telegram.ext import InlineQueryHandler

class TelegramBot:
    def __init__(self, api_key, crypto_client):
       # code from our last post ...
       
       inline_price_handler = InlineQueryHandler(self.inline_price)
       self.dispatcher.add_handler(inline_price_handler)
       
       # more code...
       
    def inline_price(self, update, context):
        query = update.inline_query.query.upper()
        if not query:
            return
        results = [InlineQueryResultArticle(
            id=query,
            title="Price",
            input_message_content=InputTextMessageContent(f'/cprice {query}')
        )]
        context.bot.answer_inline_query(update.inline_query.id, results)
```

# How does it work

We can now directly talk to the bot via `@botname <query-string>`, like below.
Note, that once we stop typing a little popup called `Price` appears, this can
be configured via `title="Price"` above:

![inline-price example part 1](/images/telegram-bot/inline-price.jpg)

Once we click on `Price`, our inline query handler basically takes the query string 
and forwards it to the `/cprice` handler:

![sad](/images/telegram-bot/inline-price02.jpg)