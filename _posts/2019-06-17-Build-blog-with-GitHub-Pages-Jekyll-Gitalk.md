---
layout: post
title:  "Build blog with GitHub Pages, Jekyll, and Gitalk"
date:   2019-06-17
categories: life-saver
tags: github-pages jekyll gitalk windows
---

* content
{:toc}

## Motivation
I wanted to set up a technical blog. A quick search on the world wide web shows that building technical blog using GitHub Pages seems to be a popular option. This article demonstrates how this blog site was built on Windows using GitHub Pages, Jekyll, and Gitalk, free of charge. I have used GitHub before but know very little about web development. This article should be suitable for readers with a similar background.



This blog is essentially a GitHub repository, where some of the repository's features are transformed via GitHub Pages, Jekyll, and Gitalk to become features of a blog site. [GitHub Pages](https://pages.github.com/) is a static site hosting service that can host webpages from a GitHub repository. [Jekyll](https://jekyllrb.com/) is a static site generator that, when used with GitHub Pages, serves to update all the pages on the blog every time a commit to the associated repository is made. [Gitalk](https://github.com/gitalk/gitalk) is a comment plugin based on the "Issues" section of the repository.



## Method

**1. Create a repository**

Create a GitHub repository and name it using the format `username.github.io`. 

E.g., if username is `steve-rogers`, the repository name should be `steve-rogers.github.io`.

**2. Set up GitHub Pages**

On the repository page, go to "Settings" and then "GitHub Pages". Choose a theme and follow the instructions. When done, the GitHub Pages should be published at `https://username.github.io/` (e.g., `https://steve-rogers.github.io/`). This is also the url of the blog site to be built.

**3. Clone the repository locally**

Clone the repository to a local folder.

The next two steps enable launching the blog site locally (shown in step 10), so that one may preview the blog site before pushing it to GitHub.

**4. Install Ruby**

Download and install Ruby with devkit [here](https://rubyinstaller.org/downloads/).

**5. Install Jekyll**

Open cmd. Type `gem install jekyll`. 

E.g.,

    C:\Users\steve> gem install jekyll

When the installation is done, type `jekyll -v` in the cmd; if the jekyll version shows up, the installation is successful. E.g.,

    C:\Users\steve> jekyll -v
    jekyll 3.8.5

**6. Download a Jekyll theme**

1. Choose a jekyll theme from [here](http://jekyllthemes.org/) and download it. 

    To download the theme, one may click "Download" directly, or click "Home" to go to the GitHub page of the theme and download the theme's repository from there. The latter option provides the latest version of the theme.
    
    This blog uses the [Cool Concise High-end theme](http://jekyllthemes.org/themes/cool-concise-high-end/); its latest version is [here](https://github.com/Gaohaoyang/gaohaoyang.github.io).
2. Go back to the local repository created in step 2 and delete all files except for the hidden `.git` folder. 
3. Copy all files from the jekyll theme folder just downloaded to the local repository.

At this step, if one follows step 10 and launches the blog site locally, the blog site would look the same as the jekyll theme demo if they downloaded the theme from the jekyll theme website, or the blog of the theme owner if they downloaded the theme from the owner's repository.

**7. Customize theme parameters**

The jekyll theme is like a blog site template. One may customize the template by filling in their own user information, such as blog title, GitHub username, email address, in the blog site's main page, header, footer, etc.

**8. Set up Gitalk**

The theme template this blog uses has a comment feature that uses Disqus. I modified the template to use Gitalk instead. First I commented out all the comment boxes in the html code built from Disqus.

1. Register an OAuth application [here](https://github.com/settings/applications/new).

    "Application name" is the name of the blog site's GitHub repository. "Homepage URL" is the url of the blog site. "Authorization callback URL" is the same as "Homepage URL". E.g.,
    ![h](_posts/OAuth-application.png)
2. Paste the following code at the end of the pages where comment function is to be enabled.
    ```html
    <!-- Gitalk comment start  -->

    <!-- Link Gitalk  -->
    <link rel="stylesheet" href="https://unpkg.com/gitalk/dist/gitalk.css">
    <script src="https://unpkg.com/gitalk@latest/dist/gitalk.min.js"></script> 
    <div id="gitalk-container"></div>     <script type="text/javascript">
        var gitalk = new Gitalk({

        // main parameters of Gitalk
            clientID: 'xxxxxxxxxxxxxxxxxxxx',
            clientSecret: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
            repo: 'username.github.io',
            owner: 'username',
            admin: ['username'],
            id: window.location.pathname,
        
        });
        gitalk.render('gitalk-container');
    </script> 
    <!-- Gitalk end -->
    ```

**9. Write blog**

**10. Launch blog locally**

**11. Push to GitHub**
