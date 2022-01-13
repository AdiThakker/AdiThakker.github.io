---
layout:     post
title:      Sync over Async Web API
date:       2021-01-20 12:32:18
summary:    This one explores how to leverage TaskCompletionSource to control the completion of a Web API request.
categories: Blogging, General
---

While working on one my recent projects, we were faced with the requirement where, when a Web Api request came in, it had to be blocked for an arbitrary amount of time, till an external event signalled for its completion and  then we could return back to its caller. 

I think the following diagram explains the situation:


In the above diagram

It's been a while since i have blogged 😞 ... have been busy with some interesting client project around serverless with messaging and hopefully will be able to blog about some learnings from it, this year!


Meanwhile, just wanted to share that i have renewed my Azure Solutions Architect certification for one more year.


![Setup]({{site.url}}/images/renew-certification.png){:height="500px" width="700px"}


You can also view my badge [here](https://www.credly.com/badges/27646698-0547-4f64-9964-a50c740ce051/public_url)


Stay Tuned!!!

