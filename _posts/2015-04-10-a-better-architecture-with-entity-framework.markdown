---
layout: post
title:  "N-Tier architecture with Web API and Entity Framework"
date:   2015-04-10 8:57:00
categories: entity-framework
---
I've been thinking about getting back into blogging lately. I've [blogged in the past][previous-blog], but I can't really remember what it was about (I think it was something about OpenGL and iOS). Now I'm doing a lot of C#, and a lot of front-end development, and I have evolved quite a bit since my iOS experiments, so I'm going to talk about all this here. Here is my first post is about setting up [Jekyll][jekyll] on Windows in order to start your own coding blog!

There are a ton of good blogging platforms out there. I used to be on [Blogger][blogger], and I had a pretty fun time with it, but it took me forever to create [Gists][gist] and copy-paste them in my posts. So, there I went looking for a blogging platform specifically for programmers, and this is where I found Jekyll. It supports markdown, code snippets, and the best part is that you can host Jekyll blogs directly on [Github][github] (which is where this blog is hosted). I've been using it for about a day, and I've had an amazing experience so far, so let's start!

# Creating your Github Page
The first thing you'll want to do is to take a look at this [guide][github-page-guide] in order to create your personal Github page. It's pretty straightforward: Create a repository that has your name in it and then clone it locally. You can test that it works by pushing an `index.html` page and you'll see that you can navigate to your newly created page. In order to maximize your experience though, you'll want to have Jekyll running locally to build and test your blog.

# Installing Jekyll and friends
Jekyll is built with Ruby, thankfully it doesn't mean that you'll have to learn to code in Ruby, but you still need to do a bit of setting up before you can get going. You'll want to read [this jekyll guide][jekyll-github-guide] right here on Github.

I'm going to recap the steps here and comment:

1. Install [Ruby][ruby]
	- This is a link to the RubyInstaller made especially for Windows users. It took me 4 clicks from Ruby's homepage to get there.
2. Install [Bundler][bundler]
	- In the folder in which Ruby was installed, you'll find a command prompt shortcut which reads "Start Command Prompt With Ruby". You can use this from now on to install gems and run other Ruby related commands. I had an error while trying to install this gem, and I basically had to install the Ruby DevKit. You will find it in the same place as the RubyInstaller, and you will need to follow [these instructions][devkit-instructions] found on the RubyInstaller wiki.
3. Install [Jekyll][jekyll]
	- In this step, it says that you should add the `github-pages` gem to a `Gemfile` in order to easily manage and update it. They forgot to mention that it's pretty easy to create this file by navigating to your local repository and running the following command: `bundle init`.

# Running Jekyll
The next part of the guide mentions that in order to run Jekyll in the same environment as Github, you should use this command: `bundle exec jekyll serve`. Basically, any Jekyll command you want to use should be preceeded with `bundle exec`.

# Creating the blog structure
A lot of the Jekyll documentation talks about the structure of the project and how you should manage your assets. To get started quickly, run the following command inside the repository: `bundle exec jekyll new my-awesome-blog`. Copy the contents of the newly created folder inside the current folder. Running this command created the basic structure of your blog, the config file, the `.gitignore` file, and it even has samples.

# Building your site
If you try to run this command now: `bundle exec jekyll build`, you will most probably get an error. The problem is that Jekyll depends on a syntax highlighter which is built with Python. So guess what? You need to install Python!

I made the mistake of installing Python 3, and later found out that Pygment (the syntax highlighter) does not support Python 3. So go ahead and install [Python 2][python-install].

If, after the installation, you still get an error like I did, then it probably means that the build process cannot find Python on your computer, so you'll want to add the path to your python installation to the [`PATH` environment variable](http://stackoverflow.com/questions/9546324/adding-directory-to-path-environment-variable-in-windows).

After all this, the build should now correctly work and the result will be in the `_site` folder.

Go ahead and run the `bundle exec jekyll serve` command and navigate to `http://localhost:4000/` to see your newly created blog!

# What now?
Now you can navigate the samples that were created in your respository and push to Github to publish your changes. If you are looking for inspiration, you can also view the source of this blog [here][github-page].

[previous-blog]: http://stevezissouprogramming.blogspot.ca/
[gist]: https://gist.github.com/
[jekyll]: http://jekyllrb.com/
[github]: https://github.com/
[blogger]: https://www.blogger.com/
[jekyll-github-guide]: https://help.github.com/articles/using-jekyll-with-pages/
[ruby]: http://rubyinstaller.org/
[bundler]: http://bundler.io/
[github-page-guide]: https://pages.github.com/
[devkit-instructions]: https://github.com/oneclick/rubyinstaller/wiki/Development-Kit
[python-install]: https://www.python.org/downloads/
[github-page]: https://github.com/ggohierroy/ggohierroy.github.io
