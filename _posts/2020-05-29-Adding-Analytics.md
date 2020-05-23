---
layout:     post
title:      Enabling Google Analytics
date:       2020-05-23
summary:    In this post, we'll look at how to enable Google Analytics for my blog. 
categories: Blogging, General
---

Since this bog is relatively new, I do not expect much traffic. Well, how can I be certain of this? May be this blog has just taken off and lots of people are already visiting it 😉.

Hmm, how can I find out more? Currently, I do not have any visibility of this data. It would be nice if I could find out information like: 

- Who’s currently visiting by blog?
- How many visitors do I get in an hour or day or a month?
- Where's the traffic coming from? 

Well, here comes [Google Analytics](https://analytics.google.com/analytics/web/provision/?authuser=0#/provision), which is a web analytics service offered by Google that tracks and reports website traffic.

OK, so how do I get this integrated?

My current [theme](https://jekyll-themes.com/mixyll/) explains how can I use Google Analytics and also more advanced [Google Tag Manager](https://marketingplatform.google.com/about/tag-manager/), (may be I can explore that in a later post).

This [link](https://support.google.com/analytics/#topic=3544906) on Google's site answers tons of questions about getting started so lets look at my steps:

1. Since I already had a gmail account, I proceeded straight to the Analytics Account Setup where I entered an account name and left the defaults checked as those are recommended. <br>
![Setup]({{site.url}}/images/Analytics-account-setup-1.png){:height="500px" width="900px"} 

 2. Selected the Web Option as that's what I was looking for. <br>
![Setup]({{site.url}}/images/Analytics-account-setup-2.png){:height="500px" width="900px"}

3. Entered relevant details about my blog site and accepted user agreements. <br>
![Setup]({{site.url}}/images/Analytics-account-setup-3.png){:height="500px" width="900px"}

4. Made note of the Tracking ID which is in the format UA-123456789-1. <br>

5. Updated my _config.yml google_analytics value with that Tracking ID number. <br>

Following is the Analytics default view that showed up after my account was created. You can see it provides several different metrics related to my blog. <br>
![Setup]({{site.url}}/images/Analytics-account-setup-4.png){:height="500px" width="900px"} 
![Setup]({{site.url}}/images/Analytics-account-setup-5.png){:height="500px" width="900px"} 
![Setup]({{site.url}}/images/Analytics-account-setup-6.png){:height="500px" width="900px"}

Cool, that was all very easy to get started... Now i'll start monitoring and see what activity get logged. 

Meanwhile I'll also explore this [link](https://support.google.com/analytics/answer/9021164?hl=en&ref_topic=3544906) to further customize and understand my analytics experience. 

In future posts I might also explore how to more information about my blog for search engines.

