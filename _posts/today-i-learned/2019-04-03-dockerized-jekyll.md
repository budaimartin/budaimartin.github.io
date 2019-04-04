---
layout: post
title:  "Dockerized Jekyll"
date:   2019-04-03 16:55:00 +0200
categories: [Today I Learned]
---

This is my first _Today I Learned_ post on this blog and what else would be a better candidate for it than Jekyll? :)

# Hello Jekyll

It's been about 20 hours since I started setting up my Jekyll blog, so I'm still learning all the cool stuff this tool can do. Nowadays I use Ubuntu 18.04 at home, so setting up Jekyll only took a few minutes and like 2 or 3 command line queries. But I also would like to take notes on my workstation where I stuck with Windows 10, and where [the setup is a bit more complicated](https://jekyllrb.com/docs/installation/windows/) and involves installation of other stuff as well.

Then I asked myself the question: **Isn't there a Docker image with Jekyll installed and set up?** (Like, for example, [Jenkins does](https://hub.docker.com/r/jenkins/jenkins/)?) Actually [there is (are)](https://github.com/envygeeks/jekyll-docker)!

<div style="text-align:center"><img src="/img/til-docker-jekyll/dockerize-all-the-things.jpg"></div>

# Hello Dockerized Jekyll!

First I started with a minimalistic Dockerfile that copies the blog into the container and starts up Jekyll.

```dockerfile
FROM jekyll/jekyll
ADD . /srv/jekyll
ENTRYPOINT ["jekyll", "serve"]
```

But that way I lost the automatic regeneration feature when I modified a file. So I decided to use the [Docker volume solution described in the readme file](https://github.com/envygeeks/jekyll-docker#usage-2).

There were some issues with the permissions, but disabling and re-enabling Docker shared volumes solved it for me. Also, under [this GitHub issue](https://github.com/envygeeks/jekyll-docker/issues/128) I found a cool docker-compose file:

```yaml
version: '3.2'

services:
  jekyll:
    image: jekyll/jekyll:minimal
    command: jekyll serve --watch --force_polling
    ports:
      - 4000:4000
    volumes:
      - .:/srv/jekyll
```

Note that I use the "minimal" version to save space and bandwidth -- it is enough for me (at least for now).

Place it in the root directory of your blog and you can fire it up with a simple `docker-compose up` command.

Also, don't forget to add the `exclude: [docker-compose.yaml]` line in your `_config.yml` so Jekyll won't copy it in your `_site` folder!
