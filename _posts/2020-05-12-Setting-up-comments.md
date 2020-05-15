---
layout:     post
title:      Setting up Commenting System
date:       2020-05-12 
summary:    I wanted to make this blog interactive, so i started looking at how to enable comments for this site. But Wait!!! This blog is based on Jekyll which is a static site generator.
categories: Blogging, General
---

If you want to setup comments for your blog, there are several options and the effort ranges from minor configuration change to complete customized setup (I think that also depends on what does your Jekyll theme). 

When i was looking for options, I was debating between [Staticman](https://staticman.net/) and [Disqus](https://disqus.com/).

I like Staticman because it lets you keep comments along with your blog and have complete control but finally decided to go with Disqus, which is third party hosted commenting system as that integration was readily available in the current theme which i am currently using for this blog. (We will Disqus 😉 about the pros & cons of this approach later in this post).

## So firstly, What is Disqus?

 It is a blog comment hosting service for web sites and online communities. The company's platform includes various features, such as social integration, social networking, user profiles, spam and moderation tools, analytics, email notifications, and mobile commenting. You can read more about them [here](https://disqus.com/).

## How does it work?

Disqus serves its contents through third-party JavaScript widgets served in an iframe element on your website. When the user visits that page, their browser will load the Disqus commenting widget. All comments are stored on Disqus’ servers, the script loads any comments stored in Disqus for the particular URL the user is on.

The service is free to use for both commenters and web sites. Web sites can pay fees to unlock additional features.

## Integration with this blog

Setting this up for my blog was fairly straight forward as [Mixyll](https://jekyll-themes.com/mixyll/) theme did provide configuration capability. 

1. Create an account with Disqus<br>
![Setup]({{site.url}}/images/disqus-account-creation.png){:height="500px" width="300px"}

2. Once logged, select I want to install Disqus on my site <br>
![Setup]({{site.url}}/images/disqus-account-setup.png){:height="500px" width="300px"}

3. Configurea new site and enter the website name. This shortname is needed for the configuration setting on my blog's _config.yml file <br>
![Setup]({{site.url}}/images/disqus-website-setup.png){:height="500px" width="600px"}

4. Modify blog's _config.yaml to include the shortname for the disqus site. <br>
![Setup]({{site.url}}/images/disqus-config-setup.png)

5. And thats it, Comments are displayed!!! <br>
![Setup]({{site.url}}/images/disqus-comments-integrated.png)

So this was very quick, in few steps we got the commenting system integrated. There are some additional settings that you can tweak to customize it further on Disqus and I'll be exploring / tweaking these after the comments start flowing on the blog 😊.

Now finally lets talk about the pros & cons of this approach since Disqus is a third party hosted commenting system.

## Pros

- Widely used and very easy to integrate

## Cons

- Comments stored on in a third-party service. This might be a big one depdending on your usage. Since this is a personal blog and i am just getting started, its not much of an issue for me.
- Limited control over the user experience, look and feel, data, and <B>privacy</B>.
- Moving your comments to a different solution could be involved as the data is hosted with third-party. 

## Final Thoughts

If using Disqus starts to become a issue, i might move to Staticman as i was keenly considering it in first place.

In the next post we will look at how to enable Google Analytics.