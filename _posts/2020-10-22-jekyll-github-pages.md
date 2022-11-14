---
title: 'Jekyll and Github pages'
author: gareth
date: 2020-10-21 00:00:00 +10:00
excerpt: Why I decided to use Github pages and Jekyll for my new blog
categories: [Blogging]
tags: [blogging]
mermaid: true
math: true
---

Earlier in my career I built my own blogging platform for my website many times. Multiple times in PHP, then some iterations were in dotnet. 

While the general recommendation is to not roll your own and to just use a free platform such as Wordpress or Medium, I think there is definitely value in building your own. Especially if you are earlier in your career, I used it as a way to practice, understand and experiment with basic website constructions.

However, fast forward 10 years I decided this time to not build my own. I've built countless blogs and websites at this point. I decided that the focus this time wasn't the blog itself but the content. I didn't even build the template this time.

### GitHub Pages

I was using GitHub pages for the previous non-blog version of my website which was a single page copy of my resume. 

I've hosted my websites many ways in the past, originally using shared PHP hosting, then actual VPS rental before going to EC2 and S3 buckets. I didn't want to do this again, too much management and too expensive for my needs. 

GitHub pages' biggest advantage is that it's free, and very easy to manage. 

![](/assets/img/2020/10/22/github-pages.png)

### Jekyll

In previous blogs I had written my own user management, post management, rss feeds (back when people cared about RSS feeds), etc. A valuable exercise at the time, I could use something like Wordpress or Ghost but that would mean ongoing costs and maintenance from me.

Enter Jekyll, with Jekyll I can write my posts using markdown in VS Code, preview them locally and then push up into Github pages when I'm done. 

I can also amend any previous posts easily, versioning and user management is handled by Github and I can just focus on not writing blog posts. 