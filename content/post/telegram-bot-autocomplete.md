+++ author = "Andrei Pöhlmann"
title = "Telegram crypto bot: autocomplete feature via prefix tree"
date = "2021-10-16"
description = "Learn how to add an autocomplete feature to your Telegram inline bot"
tags = [
"telegram",
"Python",
"tutorial"
]
categories = [
"misc"
]
+++

In this post we improve our inline Telegram crypto bot by adding an 
autocomplete feature that dispalys possible crypto symbols based on the
user's input. We will implement this autocomlete feature using a prefix 
tree. If you're not familiar with prefix trees, checkout my 
previous post [IMPLEMENT A PREFIX TREE (AKA. TRIE)](https://andrei.poehlmann.dev/post/implement-trie-python/).

Here's a preview of our final inaline bot with autocomplete:

![Inline mode](/images/telegram-bot/ac.gif)

# Prerequisites

You can catch up on our previous work in the following posts:

* [HOW-TO: (DOCKERIZED) TELEGRAM BOT THAT QUERIES FTX FOR CRYPTO PRICES](https://andrei.poehlmann.dev/post/telegram-bot-1/)
* [TELEGRAM CRYPTO BOT: ADDING INLINE MODE](https://andrei.poehlmann.dev/post/telegram-crypto-bot-inline-mode/)
* [IMPLEMENT A PREFIX TREE (AKA. TRIE)](https://andrei.poehlmann.dev/post/implement-trie-python/)

# Code

The code changes will consist of two parts:

1. Implement a trie class that provides the autocomplete feature
2. Update our Telegram bot class, i.e. the inline handler, to make
use of the new trie class
   

## 1. Trie with autocomplete

This class will basically contain the main logic for our autocomplete. 

### How does it work
The idea is as follows: 

* we populate a prefix tree with (a subset of) crypto 
symbols from FTX, e.g. `'BTT-PERP', 'BTC-PERP', 'BTC/USD', ...` and many
more (we need to implement this)
* whenever the user talks to the bot via an inline query, that 
query gets forwarded to our inline-handler (this is done by Telegram automatically, 
  so we don't have to do anything here)
* we use the query string from the previous step as a prefix to do a
  recursive search through our trie: the result of this recursive search 
  will be the autocompletion (we need to implement this)
* finally, the results from the previous step, if any, will be displayed to the
  user in the app 
  
### Show me the code already
```Python
class Node:
    def __init__(self):
        self.end = False
        self.children = {}

    def autocomplete(self, prefix):
        # we're at the end of a word, yield the result
        if self.end:
            yield prefix

        # else, recurse over each child-character
        # of the current node and append the 
        # corresponding letter to the prefix
        # -> this will build up the autocompleted string
        for letter, child in self.children.items():
            yield from child.autocomplete(prefix + letter)

class Trie:
    def __init__(self):
        self.root = Node()

    def insert(self, word):
        cur = self.root
        for c in word:
            if c not in cur.children:
                cur.children[c] = Node()
            cur = cur.children[c]
        cur.end = True

    def autocomplete(self, word):
        cur = self.root
        # starting at the root
        # traverse the trie for each 
        # character in `word`
        for c in word:
            cur = cur.children.get(c)
            if cur is None:  # word does not exist in our trie
                return
        
        # recursively autocomplete all possible words
        # starting at the final character node
        yield from cur.autocomplete(word)

if __name__ == '__main__':
    trie = Trie()
    trie.insert('foobar')
    trie.insert('foo')
    trie.insert('foob')
    trie.insert('foor')
    trie.insert('bar')

    print(list(trie.autocomplete('foo')))
```

Note, the `'__main__'` part is actually not necessary but it allows you 
to test the functionality:

```bash
apoehlmann:~/workspace/telegram-bot$ python trie.py 
['foo', 'foob', 'foobar', 'foof']
```


## 2. Update the inline handler to make use of the autocomplete feature

I will just present the whole Telegram class because I also did some 
refactoring of our original code and I think there's no simple
way of only presenting the changed parts.

### How does it work:

The main idea:

1. We add an `update_markets()` method, which queries FTX for all crypto
  data and filters out only crypto symbols that end with either `USD` or `PERP` 
   (mainly to reduce the size of the result + I'm not interested in futures and 
   the other stuff).
  We save these filtered crypto symbols into a variable called `self.market_names_perp_usd`
2. Our `TelegramBot` class now also instantiates a `Trie` object with all
crypto symbols in `self.market_names_perp_usd`
3. Whenever the user talks to the bot via an inline query, our inline handler
first fetches a list of autocompleted words, i.e. 
   ```Python 
   autocompleted = self.trie.autocomplete(query)
   ```
   If this list is non-empty we iterate through the market data and fetch 
   the prices of all coins which are part of `autocompleted`.
4. In a last step, we sort the market data alphabetically by the crypto symbol before displaying
the top 50 results to the user (50 is a constraint given by Telegram, 
   [see the API doc](https://core.telegram.org/bots/api#answerinlinequery))
   

### Show me the code already

The code can be seen in action in the GIF at the top of the page.

```Python
class TelegramBot:
    def __init__(self, api_key, crypto_client):
        self.crypto_client = crypto_client
        self.markets, self.market_names_perp_usd = [], []
        self.update_markets()

        self.trie = None
        self.init_trie(self.market_names_perp_usd)

        # Create the Updater and pass it your bot's token.
        self.updater = Updater(api_key)
        # Get the dispatcher to register handlers
        self.dispatcher = self.updater.dispatcher

        # on different commands - answer in Telegram
        self.dispatcher.add_handler(CommandHandler("cprice", self.cprice))
        self.dispatcher.add_handler(CommandHandler("cp", self.cprice))

        inline_price_handler = InlineQueryHandler(self.inline_price)
        self.dispatcher.add_handler(inline_price_handler)

        unknown_handler = MessageHandler(Filters.command, self.unknown)
        self.dispatcher.add_handler(unknown_handler)

    def update_markets(self, with_trie=False):
        self.markets = [coin for coin in self.crypto_client._get(f"markets")]
        self.market_names_perp_usd = [entry["name"] for entry in self.markets
                                      if entry["name"].endswith('PERP') or
                                      entry["name"].endswith('USD')]
        if with_trie:
            self.update_trie(self.market_names_perp_usd)

    def init_trie(self, words=[]):
        self.trie = Trie()
        self.update_trie(words)

    def update_trie(self, words):
        if self.trie is None:
            self.trie = Trie()
        for word in words:
            self.trie.insert(word)

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
        # and update the trie at the same time
        # so we add new symbols if any new ones appeared since
        # the last time we queried the exchange
        self.update_markets(with_trie=True)

        # for each coin symbol provided by in the query
        text = ''
        for coin in coins:
            # transform it to uppercase first
            # because the FTX market names are all uppercase
            coin_upper = coin.upper()
            found = False

            # for each market entry fetched from the exchange
            for entry in self.markets:
                # if there's a match with our crypto symbol
                if entry['name'].startswith(coin_upper):
                    found = True
                    # append it to our string result
                    text += f'{entry["name"]} price: {entry["last"]}\n' \
                            f'Last 01h: {100 * entry["change1h"]:.2f}%\n' \
                            f'Last 24h: {100 * entry["change24h"]:.2f}%\n\n'
                    break
            if not found:
                text += f'Sorry, no crypto symbol starting with {coin.upper()} has been found.\n\n'
        context.bot.send_message(chat_id=update.effective_chat.id, text=text)

    def inline_price(self, update, context):
        query = update.inline_query.query.upper()
        if not query:
            return
            
        # get a generator of autocompleted words 
        autocompleted = self.trie.autocomplete(query)
        if autocompleted is not None:
            ac = set(autocompleted)
            # for each crypto coin, create a InlineQueryResultArticle
            # if the name is present in the set of autocompleted symbols
            # finally, sort the result alphabetically 
            results = sorted([
                InlineQueryResultArticle(
                    id=coin['name'],
                    title=coin['name'],
                    input_message_content=InputTextMessageContent(f'/cp {coin["name"]}')
                ) for coin in self.markets if coin['name'] in ac
            ], key=lambda x: x.id)
            context.bot.answer_inline_query(update.inline_query.id, results[:50])

    def unknown(self, update, context):
        context.bot.send_message(chat_id=update.effective_chat.id, text="Sorry, I didn't understand that command.")

```

# Can we do better?

Indeed, we can. Right now, we use double for-loops where both loops iterate through lists: the symbols 
list (given by the query) and the market data. Instead of iterating through the whole market data list
to compare if any item in it matches the symbol of the query we could use a different data
structure for the market data, e.g. a set. This would allow doing existence checks in `O(1)`
runtime, making the inner loop obsolete.

Also, our autocompletion right now has no memory: for every new query we run the
whole recursive search, even when the user queries the same crypto symbol that was 
queried before. A way more likely use-case is that users query a bunch of 
coins with a high frequency, while most of the coins won't get queried at all. So
it might make sense to add a cache to the prefix tree.

Also, note that our bot right now updates the trie only in one direction: new coins/symbols
are being added to the try, while there is no purge of outdated ones that might not
be available on the market anymore. One might want to change this, else users might be
presented with autocompleted symbol names which will just lead to a notification that no cyrpto
symbol with the corresponding name was found.

These are just a few optimizations. Since the whole code base was kept simple for ease
of understanding, I'm sure there are quite a few more possible improvements.

# References

[Implementing a trie to support autocomplete in python](https://stackoverflow.com/questions/46038694/implementing-a-trie-to-support-autocomplete-in-python)