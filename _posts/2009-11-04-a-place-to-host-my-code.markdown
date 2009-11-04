---
layout: post
title: "A place to host my code"
---

I am by no means a master coder, but I think it's awesomely fun and I really like learning new things like [Cappuccino](http://cappuccino.org) and (for this blog) [jekyll](http://wiki.github.com/mojombo/jekyll/).

I have a regular blog but that's very much about everyday life, and I've been wanting to write a bit more code-specific things. And I thought that since [GitHub](http://github.com) hosts so many awesome code projects, maybe it could host my code-blog as well.

I took this theme from [Tom Ward](http://tomafro.net/) and made some slight changes to my liking, so thanks to him for the template!

Here's a quick sneak preview of what I want to write a short tutorial about:

{% highlight javascript %}
uploadButton = [[UploadButton alloc] initWithFrame: CGRectMakeZero()] ;
[uploadButton setTitle:"Select File"] ;
[uploadButton setURL:"http://localhost:3000/uploads/create"];
[uploadButton setDelegate: self];
[uploadButton setAutoresizingMask:CPViewMinXMargin | CPViewMaxXMargin];
[uploadButton setBordered:YES];
[uploadButton sizeToFit];
[uploadButton setCenter:CGPointMake(CGRectGetWidth([contentView bounds])/2.0, 470)];
[contentView addSubview:uploadButton];
{% endhighlight %}

If you find anything useful or want to encourage me to continue writing, please comment! =)