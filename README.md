This blog is base on Chirpy Jekyll Theme, please check the following pages if you need additional informations:  
 - [Jekyll](https://jekyllrb.com/)
 - [Chirpy theme](https://github.com/cotes2020/jekyll-theme-chirpy)


## Contributing

You can either:
 - create a fork of the repository and work on your version and then make pull requests once your article is done
 - ask me to add you as a contributor so that you can directly push into master


## Local testing

If you want to test the blog locally to see how your article will render (environnement includes live reloading), just follow the jekyll installation process:
 - install `gem` which is a package manager for Ruby
 - run `gem install jekyll bundler`
 - go inside your local dir and run `bundle` which will install all the dependencies
 - now you are ready to serve by running `./tools/run.sh` or `bundle exec jekyll s`

## Writing an article

Writing a new article is easy: just create a new file in the "_posts" directory with the following name: "YYYY-MM-DD-TITLE.md".
Then the first block of the file is the header and must contain informations about the article, as:
```
---
title: How to make a real fast replay mode
author: Timothée Pecatte
date: 2021-11-18 23:33:00 +0800
categories: [Tips]
tags: [replay,front]
pin: true
---
```

## License

This work is published under [MIT](https://github.com/cotes2020/jekyll-theme-chirpy/blob/master/LICENSE) License.
