---
layout:     post
title:      Setting up Commenting System
date:       2020-05-15
summary:    This post explores how I enabled comments for my blog
categories: Blogging, General
---

If you want to setup comments for your blog, there are several options and the effort ranges from minor configuration change to complete customized setup (I think that might also depend on what Jekyll theme you use as several themes support different integrations). 

When i was looking for options, I was debating between [Staticman](https://staticman.net/) and [Disqus](https://disqus.com/).

I like Staticman because it lets you host comments on your blog and you can have complete control over it but eventually decided to go with Disqus, which is third party hosted commenting system as that integration was readily available in the [current](https://jekyll-themes.com/mixyll/) theme. (We will Disqus 😉 about the pros & cons of this approach later in this post).

## So, What is Disqus?

 It is a blog comment hosting service for web sites and online communities. The company's platform includes various features, such as social integration, social networking, user profiles, spam and moderation tools, analytics, email notifications, and mobile commenting. 

## How does it work?

Disqus serves its contents through third-party JavaScript widgets served in an iframe element on your website. When the user visits your page, their browser will load the Disqus commenting widget. All comments are stored on Disqus’ servers, the script loads any comments stored in Disqus for the particular URL the user is on.

The service is free to use for both commenters and web sites. There are different [pricing](https://disqus.com/pricing/) options and Web sites can pay fees to unlock additional features.

## Integration with this blog

Setting this up for my blog was fairly straight forward as my [theme](https://jekyll-themes.com/mixyll/) did provide that configuration option. Following are the steps i followed:

1. Create an account with Disqus.<br>
![Setup]({{site.url}}/images/disqus-account-creation.png){:height="500px" width="300px"}

2. Once logged, select - I want to install Disqus on my site. <br>
![Setup]({{site.url}}/images/disqus-account-setup.png){:height="400px" width="300px"}

3. Configure new site and enter the website name. This shortname is needed for the configuration setting in the blog's _config.yml. <br>
![Setup]({{site.url}}/images/disqus-website-setup.png){:height="500px" width="600px"}

4. Modify _config.yml to include the shortname for the disqus site. <br>
![Setup]({{site.url}}/images/disqus-config-setup.png)

5. And thats it, Comments are displayed!!! <br>
![Setup]({{site.url}}/images/disqus-comments-integrated.png)

So in just few steps I got the commenting system integrated. There are several additional settings that you can tweak to customize further on Disqus site and I'll be exploring those once comments start flowing on the blog 😊.

Now lets talk about some of the pros & cons of this approach since Disqus is a third party hosted commenting system.

## Pros (partial list):

- Widely used.
- Very easy to setup.
- Responsive  & mobile friendly.
- Well integrated with other social media platforms and so eliminates the need to create new accounts on every blog. Sharing comments on these platforms is also very easy.

## Cons (partial list):

- Comments stored on in a third-party service. This might be a big one depending on your usage. Since this is a personal blog and i am just getting started, its not much of an issue for me.
- Limited control over the user experience, look and feel, data, and <B>[privacy](https://disqus.com/data-sharing-settings/)</B>. As a Disqus user, you might want to opt out of it.

## Final Thoughts:

I gave Disqus a try because of its simple setup. However if privacy becomes an issue going forward, i might consider Staticman as that was other self-hosted option i was considering.