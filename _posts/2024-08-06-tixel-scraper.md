---
title: Scraping Tixel to notify you instantly of tickets on sale
date: 2024-08-08 18:00:00 PM
categories: [Python, Automation]
tags: [python, web-scraping, tixel, telegram]     # TAG names should always be lowercase
description: Missed out on tickets to your favourite band? Utilise this tixel web-scraper to never miss out again!
image:
   path: /assets/img/posts/tixelscraper/preview.png
   alt: Tixel Web Scraper
comments: true
# 1200 x 630
---

## The Problem
Tixel is a ticket resale service in Australia which operates as a middle-man to ensure that those selling tickets online have legitimate tickets and that those same sellers get their money. It's a great service for buyers and seller alike however in my experience it's notification system for when a ticket is available is abyssmal. 

Earlier this year when I wanted to attend [Bring Me The Horizon's](https://www.youtube.com/watch?v=Jq4NhAnnD0Q){:target="_blank"} Rod Laver Arena event (a show where almost 1000 fans were on the waitlist) I found that Tixel's email notification system was delayed for as long as 15 minutes after a ticket was posted online.

Compounding the issue is Tixel's own processes:
> We send out alerts in randomised order every few seconds, until that ticket is purchased. For popular events where tickets are bought quickly, there is not enough time to send out a notification out to everyone.

Hence the purpose of this blog-post, let's come up with a way to circumvent their system to be notified instantly when a ticket becomes available.

![notification](/assets/img/posts/tixelscraper/successful_notification.png){: width="243" }

## Requirements

| Requirement | Comment |
|-------|--------|
| python | We will use python as our codebase |
| IDE of choice| I use [PyCharm](https://www.jetbrains.com/pycharm/download/){:target="_blank"} | 
| Telegram Android/iOS Application | This is the method we will use to notify us a ticket is available|

#### Install the following python packages via your IDE or pip install:

```python
pip install playwright
pip install requests
```

## Getting Started
We will be utilising [Playwright for Python](https://playwright.dev/python/docs/intro){:target="_blank"} as our way to scrape the tixel website.

1. Install playwright via pip install or your IDE's package manager
2. Create a new python file and name it (e.g. `tixel_scraper.py`)
3. If this is your first time installing playwright you may need to install the necessary browser binaries. Open your console/terminal and run the following command:

```sh
python -m playwright install
```

> You may need to use `python3` instead of `python`. 
> Alternatively you may need to specify your python installation path if it is not able to find playwright.
{: .prompt-tip }
```sh
"C:\path\to\python.exe" -m playwright install
```
If this is still not working perhaps you are using a virtual environment. If so try the following in your console/terminal:

```sh
.\venv\Scripts\activate
python -m playwright install
```

## Writing Our Code

1. Let's import the modules we're going to use
   ```python
   from playwright.sync_api import Playwright, sync_playwright, TimeoutError
   import re
   import requests
   import time
   ```
2. Now let's write a function to open the website and call that function

   ```python
   def run(playwright: Playwright) -> None:
      browser = playwright.chromium.launch(headless=False)
      context = browser.new_context()
      page = context.new_page()

      try:
         url = "https://tixel.com/au/music-tickets/2024/09/22/parkway-drive-john-cain-arena-me"
         page.goto(url)
         page.wait_for_load_state("load")
      finally:
        context.close()
        browser.close()

   with sync_playwright() as playwright:
      run(playwright)
   ```
   > Note we are running the chromium browser with `headless=False` for now so we can see what is happening. We will change this at the end.
   {: .prompt-info }

3. When we run the code we should see the page loading in the chromium browser and then close once it's fully loaded
![example](/assets/img/posts/tixelscraper/first_example.png)


4. Now let's extract some data from the website! At the time of writing tixel will group similar tickets into individual cards based on their type (e.g. seated vs. general admission):
![cards](/assets/img/posts/tixelscraper/ticket_grouping.png){: width="504" }

    So my approach is simple, usually I like to be in the General Admission area and these are quite often the sought after tickets, so we will specify that we only want to extract data from the General Admission cards.
      Let's add the following after the `page.wait_for_load_state("load")` line of our run function:
   ```python
         try:
            button = page.wait_for_selector('button:has-text("General Admission")', timeout=10000)
            ticket_details = button.inner_text()
            print(ticket_details)
         except TimeoutError:
            print("No General Admission Tickets for sale. Retrying in 20 seconds.")
            return
   ```
5. After we execute the code we should get a print out in our console (assuming there is atleast one ticket available for the show in this category) which displays the following:
   ```
   GENERAL ADMISSION STANDING 18+

   10+ tickets available

   From $75

   Process finished with exit code 0
   ```
6. Next we will use regular expression we can extract the price of the ticket. Add the following below the `print(ticket_details)` line of our run function:
   ```python
   cost = int(''.join(re.findall(r'(?<=\$)[0-9,]+', ticket_details)))
   print(cost)
   ```
   If a ticket is availabe I will get the price of the ticket printed out in the console:
   ```
   75
   ```

7. Lets write our message output in a nice format with a link to the event so when we are notified we can quickly jump in and purchase the ticket. This can sit outside our second try block:
   ```python
   output = "Ticket Price = ${}, {}".format(cost, url)
   ```

8. Here is the full code so far:
```python
   from playwright.sync_api import Playwright, sync_playwright, TimeoutError
   import re
   import requests
   import time


   def run(playwright: Playwright) -> None:
      browser = playwright.chromium.launch(headless=False)
      context = browser.new_context()
      page = context.new_page()

      try:
         url = "https://tixel.com/au/music-tickets/2024/09/22/parkway-drive-john-cain-arena-me"
         page.goto(url)
         page.wait_for_load_state("load")
         try:
               button = page.wait_for_selector('button:has-text("General Admission")', timeout=10000)
               ticket_details = button.inner_text()
               print(ticket_details)
               cost = int(''.join(re.findall(r'(?<=\$)[0-9,]+', ticket_details)))
               print(cost)
         except TimeoutError:
               print("No General Admission Tickets for sale. Retrying in 30 seconds.")
               return
         output = "Ticket Price = ${}, {}".format(cost, url)

      finally:
         context.close()
         browser.close()


   with sync_playwright() as playwright:
      run(playwright)
```

## Notification Setup

1. Before we add our code to send the notification we need to do some setup with our Telegram app. If you haven't already, download the [Telegram app](https://telegram.org/apps) and sign up for an account.

2. Contact the [BotFather](https://telegram.me/BotFather) bot and send the command `/start` followed by `/newbot`. Follow the instructions sent by BotFather to setup the bot, name it as you see fit (e.g. TixelNotificationBot).

3. Once your bot is created you will receive an API key, copy this value and store it somewhere for later. 

4. Start a new message and locate your bot via its name:

    ![bot-message](/assets/img/posts/tixelscraper/tixelNotificationBot.jpg){: width="180" height="300" }

5. Send your bot the `/start` command.

6. Lastly we need to find our own ID on Telegram, to do this you can message the [IDBot](https://telegram.me/myidbot) and send `/start` to receive your ID value. We will use this in the next section.

## Adding the Notification Function to python

1. Since we can now extract the data for the ticket we want, let's implement the way to notify us of the tickets availability. Create two new variables before our run function:
```python
token_file_path = 'keys/telegram_token.txt'
chat_id_file_path = 'keys/telegram_chat_id.txt'
```

3. Create a folder in your `tixel_scraper.py` directory called `keys` and create two text files named `telegram_token.txt` and `telegram_chat_id.txt`. 

4. Open these files and save your bot API key from earlier in the `telegram_token.txt` file and save your Telegram ID in the `telegram_chat_id.txt` file.

5. Add the following code so we can read the text files, this needs to go after the variable declaration but before the run function:
```python
   with open(token_file_path, 'r') as file:
      token = file.read().strip()

   with open(chat_id_file_path, 'r') as file:
      chat_id = file.read().strip()
```

5. Let's add the following if statement to our run function just below our `output = "Ticket Price = ${}, {}".format(cost, url)` line. This firsts checks the cost of the ticket matches the threshold. If true it will then send a notification to our Telegram bot who will then in turn notify us:
   ```python
      if cost <= 75:
         print("Notification Sent")
         message = output
         url = f"https://api.telegram.org/bot{token}/sendMessage?chat_id={chat_id}&text={message}"
         print(requests.get(url).json())  # this sends the message
         time.sleep(300)  # Pause for 5 minutes to reduce spam (300 seconds)
      else:
         print("No ticket found for the minimum price, trying again in 20 seconds....")
   ```
   > I like to set a cost threshold particularily if I am trying to get a cheap ticket to an event that is not selling well. If your event is very popular and the tickets are being purchased immediately after listing it may be worth setting the price threshold to a large number (e.g. 500) so that you are notified of every ticket on sale. 
   {: .prompt-info }

7. We want this to run repeatedly until we stop the script so let's write a new function to replace our existing `sync_playwright` call. And make sure we call the function at script start.
```python
   def run_every_20_seconds():
      while True:
         with sync_playwright() as playwright:
            run(playwright)
            time.sleep(20)
   
   run_every_20_seconds()
``` 

8. Lastly make sure you set `headless=True` in our browser variable so that the Chromium browser does not render on the screen.

## Finishing Up

And that's all the coding done, run the script now and you should now be getting notifications if your ticket type and price are available. Note for testing purposes you may want to alter the ticket type and/or cost to ensure you are receiving notifications prior to running this script for the correct parameters. 

![notification](/assets/img/posts/tixelscraper/successful_notification.png){: width="243" }

Here is the finished code from this post:
```python
from playwright.sync_api import Playwright, sync_playwright, TimeoutError
import re
import requests
import time

token_file_path = 'keys/telegram_token.txt'
chat_id_file_path = 'keys/telegram_chat_id.txt'

with open(token_file_path, 'r') as file:
    token = file.read().strip()

with open(chat_id_file_path, 'r') as file:
    chat_id = file.read().strip()


def run(playwright: Playwright) -> None:
    browser = playwright.chromium.launch(headless=True)
    context = browser.new_context()
    page = context.new_page()

    try:
        url = "https://tixel.com/au/music-tickets/2024/09/22/parkway-drive-john-cain-arena-me"
        page.goto(url)
        page.wait_for_load_state("load")
        try:
            button = page.wait_for_selector('button:has-text("General Admission")', timeout=10000)
            ticket_details = button.inner_text()
            cost = int(''.join(re.findall(r'(?<=\$)[0-9,]+', ticket_details)))
        except TimeoutError:
            print("No General Admission Tickets for sale. Retrying in 20 seconds.")
            return
        output = "Ticket Price = ${}, {}".format(cost, url)
        if cost <= 75:
            print("Notification Sent")
            print(output)
            message = output
            url = f"https://api.telegram.org/bot{token}/sendMessage?chat_id={chat_id}&text={message}"
            print(requests.get(url).json())  # this sends the message
            time.sleep(300)  # Pause for 5 minutes (300 seconds)
        else:
            print("No ticket found for the minimum price, trying again in 20 seconds....")
    finally:
        context.close()
        browser.close()


def run_every_20_seconds():
    while True:
        with sync_playwright() as playwright:
            run(playwright)
        time.sleep(20)


run_every_20_seconds()
```

## How could this be improved?
Tixel actually has an auto purchase option for some events, if this is active for your event I recommend utilising it as well as this script.

That said for events where auto purchase is disabled we could set up an auto purchase function by having us login to Tixel with a saved credit card and purchase the ticket once it's available. I am happy to do a manual purchase myself but if you really wanted that ticket you could have a go at implementing this.

Another improvement would be to search all the available cards, currently we are only looking at the first button which matches the name:
```python
button = page.wait_for_selector('button:has-text("General Admission")', timeout=10000)
```
Ideally we could look at all the available buttons and do some regex matching which meet our parameters. I was a bit too lazy to implement this as my crude method seemed to work okay.

Thanks for reading.

Let me know your thoughts in the comments below!

---

