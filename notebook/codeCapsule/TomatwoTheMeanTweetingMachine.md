# Tomatwo, the mean tweeting machine

> [Tomatwo, the mean tweeting machine | Code Capsule](https://codecapsule.com/2011/02/20/tomatwo-the-mean-tweeting-machine/)

I have seen many blog posts about Twitter bots, but they all seem pretty useless to me. People have this tendency to overuse things like Markov chains or other conversational agent library, just to make their bot look cool. But what’s the point of all that, and what problem are they solving? My main issue with Twitter is that it takes time. If one intends to create a presence on Twitter, then the time to invest in that is non-trivial. You have to post almost every day, and regularly throughout the day, which means you have to stop your current activity, connect to Twitter, write something, and then resume your activity. The “stop/resume” part, multiple times per day, is *not* cool, but not cool at all. So I came up with a cheap trick to fake on-line presence using Twitter.

## What I want a Twitter bot to do (Requirements)

1. Given a parameter representing a number of tweets per day, I want the bot to post *around* that many tweets, some days more, some days less (or even zero), so it fakes what a real person would do.
2. I want to give the bot a list of RSS feeds, and the bot should be able to post links from these feeds.
3. I want to give the bot a list of messages, and the bot will tweet them later. The idea here is that I can write 30 tweets in 10 minutes once a week, and then the bot is going to disseminate those 30 tweets over the course of a week, without any further intervention from me. So the time I spend on dealing with Twitter is really 10 minutes per week (not including answers to private messages). I also would like a web interface to the list of tweets so I can add/remove tweets just using a web browser.

Note that there might be web apps already doing all that, but I am not okay with giving them my credentials. I am going to make a Python script so I can keep all that on my own server.

## What I have so far

Given the requirements above, I came up with Tomatwo, a small Twitter bot that implements requirements #1 and #2 from the above list. For the number of tweets, it simply uses a normal distribution to define a number of tweets per day. Every day around midnight, the **planner** is run and determines when tweets will be posted: this is a list of dates (date = day, hour, minute) at which tweets will be posted during the day. As for RSS feeds, the **crawler** is run every right before the planner. It reads the RSS feeds, and stores the feeds’ titles, authors, dates, and URLs. Then, every N minutes (with N within [1, 5]), the **checker** is run. The checker verifies the list of hours that was made by the planner, and if the first date is equal or lower to the current date, then a RSS feed is taken out of the database, a message is made using the title and URL of that feed, and this message is tweeted! Also, the link is shortened using bit.ly so that everything fits within the 144-character limit.

As for requirement #3, I have not yet implemented it. The code, under BSD license, is available here: [https://github.com/goossaert/tomatwo/](https://github.com/goossaert/tomatwo/). The README file contains a step-by-step guide to the installation and use Tomatwo.

Drop me a comment or an email if you have any question/feedback.

Enjoy!
