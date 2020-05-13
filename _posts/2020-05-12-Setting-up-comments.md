---
layout:     post
title:      Setting up comments
date:       2020-05-12 
summary:    I wanted to make this blog interactive, so i started looking at how to enable comments for this site. But Wait!!! This blog is using Jekyll which is a static site generator.
categories: Blogging, General
---

If you want to setup comments for your blog, there are several options ranging from:

- [Static commenting](https://tlvince.com/static-commenting)
- [Self hosted]()
- [Third party hosted]()

I decided to go with [Disqus](https://disqus.com/) which is third party hosted commenting system as that integration was readily available in the current theme which i am currently using for this blog. (We discuss about the pros & cons about this later in this post).

## What is Disqus?

 It is a blog comment hosting service for web sites and online communities that use a networked platform. The company's platform includes various features, such as social integration, social networking, user profiles, spam and moderation tools, analytics, email notifications, and mobile commenting. You can read more about them [here](https://disqus.com/).

## How does it work?

Disqus serves its contents through third-party JavaScript widgets served in an iframe element on your website. When the user visits that page, their browser will load the Disqus commenting widget. All comments are stored on Disqus’ servers, the script loads any comments stored in Disqus for the particular URL the user is on.

Disqus operates with different The service is free to use for both commenters and web sites. Web sites can pay fees to unlock additional features.

## Integration with this blog

Setting this up for my blog was fairly straight forward as [Mixyll](https://jekyll-themes.com/mixyll/) theme did provide configuration capability. 

1. I created an account with Disqus <br>
![Setup]({{site.url}}/images/disqus-account-creation.png)

2. Once that was done, I Select 2nd option - I want to install Disqus on my site <br>
![Setup]({{site.url}}/images/disqus-account-setup.png)

3. Configured new site and entered the website name. This shortname was needed for the configuration setting on my blog's _config.yml file <br>
![Setup]({{site.url}}/images/disqus-website-setup.png)

4. Modified _config.yaml section to include the shortname for the disqus site. <br>
![Setup]({{site.url}}/images/disqus-config-setup.png)

5. And thats it, Comments integrated!!! <br>
![Setup]({{site.url}}/images/disqus-comments-integrated.png)

Very quick to get started. There are some additional settings that you can tweak to customize it further and I'll be exploring / tweaking these after the comments start flowing on the blog 😉.

OK, so that was it with enabling comments for my blog.

Alright now, since Disqus a third party hosted commenting system, its good to know about the pros & cons:

## Pros

- Widely used and very easy to integrate

## Cons

- Comments stored on in a third-party service. This might be a big one depdending on your usage. Since this is a personal blog, its not much of an issue for me.
- Limited control over the user experience, look and feel, data, and <B>privacy</B>.
- Moving your comments to a different solution could be involved as the data is hosted with third-party. 

## Other Considerations

