---
layout: post
title: "Migrating from blogger to octopress (done!)"
date: 2014-08-24 19:29:46 +0200
comments: true
published: true
categories: 
- octopress
- blog
- python
---
It feels a bit like this ![Light bulb](http://i.imgur.com/t0XHtgJ.gif)

but when I had to write a new blogpost and had to embed some code snippets into my blogger hosted blog, I got so pissed of that I chose to migrate my existing blog to a github hosted instance of octopress. 

Jason Jarrett made a pretty nice tutorial about the whole process [here](http://staxmanade.com/2014/04/migrating-blogspot-to-octopress-part-1-introduction/). 
However, I had to change a couple of things and I thought those might be helpful to anybody who faces the same problems.

The overall process is something like:

+ setup octopress
+ import an exported dump of blogger
+ fix internal links
+ setup redirects from blogger to your new blog

##Setting octopress up
Nothing to say here. Just go to octopress website and follow the instructions.

##Importing posts from blogger
This is pretty straightforward too. Export your blogger content as xml, and use [this ruby script](https://gist.github.com/juniorz/1564581) that generates a _posts_ folder containing all the posts exported from blogger. Anyway, refer to [Jason's blog](http://staxmanade.com/2014/04/migrating-blogspot-to-octopress-part-4-import-content-into-ctopress/), everything is described accurately there.

##Fix internal links
You'll have to play a bit with _sed_ in order to fix internal links, otherwise they will keep pointing to blogspot. 

## Setup redirects
Here is where things get interesting, because you will want to make visitors of the old url be redirected to the new blog (possibily with 301). The whole process is a bit tricky due to blogger limitation (for more details check [here](http://staxmanade.com/2014/04/migrating-blogspot-to-octopress-part-6-301-redirect-old-posts-to-new-location/)). 

There are a couple of other obstacles too: the script available is powershell only. Moreover, Jason suggets to use the alias plugin [which seems to be broken at the moment](https://github.com/imathis/octopress/issues/1610).

However, [jekyll-redirect](https://github.com/jekyll/jekyll-redirect-from) seems to work fine, so I chose to use it in my solution.

And finally, since powershell is not an option, here is my python version of the script. It binds the post id with the title of the posts, loop all the html files in your post folder, injects the redirection in the yaml header.

Using it is as easy as calling:
```sh
python blogger_import.py -p octopress/source/_posts/ -b ./blog-08-22-2014.xml
```

where p is the path of your posts folder, b is the xml file produced by blogspot.

Here is the script:

{% gist e46635e3d7de475b0546 %}


## TL;DR

+ Read [Jason's blog](http://staxmanade.com/2014/04/migrating-blogspot-to-octopress-part-1-introduction/)
+ When setting up redirection, install [jekyll-redirect](https://github.com/jekyll/jekyll-redirect-from)
+ Use my script to inject redirection in the header of exported blogposts
+ Setup blogger template as described in Jason's blog

PS: I still need to write the blogpost that made me switch to octopress.
