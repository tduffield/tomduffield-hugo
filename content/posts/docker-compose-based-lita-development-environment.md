---
title: "Docker Compose Based Lita Development Environment"
date: 2017-03-13
tags: [ programming ]
---

In the continuation of the Lita development that I've been doing recently, I've found myself needing a more sophisticated development environment for Lita. Running `bundle exec lita` locally just was not cutting it anymore. My Lita plugin was starting to pepper my hard-drive with various files and the number of services I was needing to manually install and manage on my local machine was increasing. Unfortunately, the Lita plugin that inspired this work is not currently open source, but I've managed to replicate the setup with a Lita plugin that I have open-sourced: [lita-announce](https://github.com/tduffield/lita-announce).

So the foundation of this setup is the [docker-compose.yml](https://github.com/tduffield/lita-announce/blob/master/docker-compose.yml), which contains two services: Lita and Redis.

```
version: '3'
services:
  lita:
    build: .
    links:
      - redis:redis
    volumes:
      - ./:/app
      - ./.docker_bundle:/app/.bundle
    environment:
      - SLACK_TOKEN
  redis:
    image: redis
```

Now, Lita already has a [docker image](https://hub.docker.com/r/litaio/lita/) that you can use to run Lita, but their image is intended for "production" cases. I needed something focused on development, which is where my [customized Dockerfile](https://github.com/tduffield/lita-announce/blob/master/Dockerfile) came in.

```
FROM ruby:2.3
MAINTAINER Tom Duffield <tom@chef.io>

RUN apt-get update && \
    apt-get install -y git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

RUN gem install bundler && mkdir /app
COPY start /start
WORKDIR /app
CMD ["/start"]
```

This Dockerfile is very similar to the one for `litaio/lita`, but with a few differences. First, it is based off the latest Ruby 2.3 version rather than 2.3.0 (which has some known bugs). Second, it has a [customized start script](https://github.com/tduffield/lita-announce/blob/master/start) that executes the lita process via [guard-process](https://github.com/guard/guard-process).

```
bundle install --path /var/bundle --jobs $(nproc) --clean
exec bundle exec guard
```

The [Guardfile](https://github.com/tduffield/lita-announce/blob/master/Guardfile) that I used watches the lita_config.rb file, as well as the entire `lib/` directory of my plugin, for file changes and restarts the Lita process if any changes are necessary.

```
interactor :off
guard "process", name: "Lita", command: "bundle exec lita" do
  watch("lita_config.rb")
  watch(%r{^lib/*})
end
```

Because I'm [syncing my plugin directory directly into the /app folder on my container](https://github.com/tduffield/lita-announce/blob/master/docker-compose.yml#L9), any changes that I make locally are immediately available in the container. The end result is that the Lita service in my container will restart anytime I make a change to a file locally. This allows me to iterate very quickly when attempting to functionally test my plugin without having to manually restart my process.

If you want to see all of the changes that were necessary, you can view them all in [this commit](https://github.com/tduffield/lita-announce/commit/52c5119f17be33858a75b2243052b6c52d480854).