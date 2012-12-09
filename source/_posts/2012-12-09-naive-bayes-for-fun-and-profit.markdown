---
layout: post
title: "Naive Bayes for fun and profit"
date: 2012-12-09 16:20
comments: true
categories: [Programming, Python, ML]
---

Five years ago, I made a website in Django for a friend in Switzerland - a website for a skiing club encouraging the sport as an activity for young children to take up, offering heavily discounted training and instruction to children under a certain age. A lovely idea I was keen in helping out with.

When I first wrote the site, the club decided they wanted to have a guestbook. Something that sounds a bit anachronistic in a world of tweets and wall messages, but it was just meant as somewhere for the kids to communicate with the team and each-other without the complexity of a forum.

Like with any user-submitted page, when I first put the site online, spam on the guestbook was an issue.

I tried to keep my approach to spam filtering simple. Asking for the sum of 2+2 seemed to work flawlessly to cut out all spam. At least it was for a while. Recently, the site has been plagued by by spam. This is an especially thorny issue given young children would make regular visits to the site.

I replaced my simple static reverse turing test with reCAPTCHA. Tested it. Everything seemed fine for a few weeks. Then a storm of spam landed on the site.

I sat down and had a think about how I could solve this problem. I could see a few options:

- Blacklist IPs of the spammers: Subscribing to a blacklist provider and filtering based on the banned addresses. 
- Check source of post: We could go a step further - all genuine visitors were kids or club members from Switzerland. We could just stop any user outside of Switzerland from posting messages (but still allow them to view the site).
- Filter based on message content: The spam messages were written in english with common keywords, and loads of (escaped) markup. The genuine messages were written in french with a rather limited word set (“salut”, “merci”, etc.)

It seemed to me that the easiest, lowest maintenance, most reliable, and most interesting to implement solution was the third one.

In fact, this seemed like a perfect case for a Naive Bayes filter. Sadly, these days, there’s nothing innovative or exciting about a Naive Bayes filter. But it still a remarkable technology. Especially in its simplicity and mathematical elegance.

I wanted to make this an interesting side project to work on. Not wanting to spend a weekend losing myself in researching a way of improving on Naive Bayes, I sought the simplest way I could provide Naive Bayes spam filtering, as a service.

Now... there’s plenty already out there. The site is written in Django, so Django [SpamBayes](http://code.google.com/p/django-spambayes/) comes to mind. There are also 3rd party services like [Akismet](http://akismet.com/).

{%pullquote%}
{"I just wanted to have a go writing a Naive Bayes calssifier service that would run in memory, and be used from any application on my server via RPC."}
{%endpullquote%}

I wrote a stand-alone python application that reads all files placed in a ham and spam directory, and trains an NLTK NaiveBayes classifier based on those files. Where each file represents a document, or “post”.

After installing, this software can be run with the command ```classify_server --ham=[DIRECTORY CONTAINING HAM FILES] --spam=[DIRECTORY CONTAINING SPAM FILES]``

I also wrote a library that provides a wrapper to the RPC calling via pika.

```
classifier = NaiveBayesDaemonClient()
classifier.is_message_spam(“Hello”)
```

Easy. I’ve deployed it to the site. Let’s see if the spam stops!

It can be found [on GitHub here](https://github.com/roderickhodgson/NaiveBayesSpamDaemon), as a nice python package.

