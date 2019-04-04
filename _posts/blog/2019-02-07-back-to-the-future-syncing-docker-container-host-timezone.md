---
layout: post
title:  "Back to the Future"
date:   2019-02-07 20:36:47 +0200
categories: Blog
---
# Handling different timezones between Docker container and host
![Back to the Future](https://upload.wikimedia.org/wikipedia/commons/thumb/2/23/Back-to-the-future-logo.svg/1000px-Back-to-the-future-logo.svg.png)

_You can also read this blog post on the [Hotels.com Technology Blog](https://medium.com/hotels-com-technology/back-to-the-future-af4431aa6e97)_

## Introduction  
Back in the summer, as Java 8 was being killed, we decided to create our new services in Java 10. The excitement of using a new version of a programming language, however, comes mixed with nervousness caused by all kinds of mysterious errors, mainly due to outdated and not-anymore-compatible plugins and dependencies; automated and "copy-paste" subtasks may not work anymore.

In this blogpost, I share a story with you about one of these mysterious errors. My goal is not to give you a chicken soup recipe for a given problem, but to share my journey I walked through during my investigation and some interesting facts I found out in the meanwhile.

## Back to the Future  
After I finished migrating one of our services to Spring Boot 2 and Java 10, the testers realized that there were no logs present according to our Splunk log-indexer. To be more precise: the old logs were there, but after sending a new request to the service, we couldn't see any new log entries. It was working fine in local and with all of our other services on the test environment, so we first thought it was some Splunk issue. So we requested an investigation from the observability team, and the results were more than unexpected: **the new log entries were in the future**!

<div style="text-align:center"><img src="https://thumbs.gfycat.com/LiquidGloomyGroundbeetle-max-1mb.gif"></div>

Checking the log files and the time on the server revealed a difference between the timezone of the JVM and the server: JVM appeared to be using GMT while the server time was in CDT. Time difference between GMT and CDT is 5 hours which was reflected in the logs (see below).

```bash
$ tail service.log
> INFO  com.hotels.[..] 2018-08-27 06:58:16,029  - Service called with params=[..]
> INFO  com.hotels.[..] 2018-08-27 06:58:16,917 [..] - Response from client: [..]
> INFO  com.hotels.[..] 2018-08-27 06:58:16,934 [..] - Service call was successful. Response=[..]
$ date
> Mon Aug 27 02:35:41 CDT 2018
```

In other, well-functioning services, the time in Splunk and the log timestamp always show a difference -- in Budapest (where timezone in summer is GMT+2), it's 7 hours. However, in those cases the time identified by Splunk is the real time (i.e. the local time when the log was created), while the timestamps in the log entries are in the past. For some reason, in our new service, we had the opposite: the timestamp was the real time, while "Splunk-time" was 7 hours in the future.

## Timezones in Java  
<div style="text-align:center"><img src="http://www.monkeyuser.com/assets/images/2018/85-going-global.png" width="500"></div>

To understand how Java handles timezones, I reached out for the Javadoc page of the `TimeZone` class, especially the [getDefault](https://docs.oracle.com/javase/10/docs/api/java/util/TimeZone.html#getDefault()) method, as we didn't have any explicit timezone settings in the service. The algorithm that decides the default timezone goes like this:

 > * Use the `user.timezone` property value as the default time zone ID if it's available.
 > * Detect the platform time zone ID. The source of the platform time zone and ID mapping may vary with implementation.
> * Use GMT as the last resort if the given or detected time zone ID is unknown.

As we didn't set the `user.timezone` system property, it was obvious that something goes wrong during the detection of the platform timezone: GMT was either resolved as platform timezone or was fallen back to.

## Digging into runtime configuration
I took a closer look to the runtime configuration of the service, and realized that the logs are saved on the host machine via a mounted volume. Those who are not familiar with [Docker volumes](https://docs.docker.com/storage/volumes/): it is possible to attach resources such as files and directories from the host to a given path in the container.

When the container is started with the `-v /usr/share/logs:/usr/share/service/logs` switch, it means that when the `/usr/share/service/logs` path is accessed in the container, it will see or modify the content of the ` /usr/share/logs` directory _on the host_. So the logs are created inside the container in GMT timezone, but persisted outside on the host in CDT timezone.

We used a similar approach of time synchronization before. With `-v /etc/localtime:/etc/localtime:ro`, the path that contain the time settings in Linux is mounted as well (note that `:ro` means, it's mounted in read-only mode). _So why isn't it working now?_

Actually, I found out that there is an `/etc/timezone` file as well, containing the canonical name of the currently set timezone. Maybe Java 10 uses another algorithm to detect timezones? Anyways, mounting `/etc/timezone` to the host means that it would use the same timezone of the host, right? Actually, **it made the container unable to start**.

<div style="text-align:center"><img src="https://cdn-images-1.medium.com/max/800/1*QZ1BMbxGyHCuprtYymJjSA.gif" align="center"></div>

## Linux vs Linux vs Linux  
Checking the contents of `/etc/timezone` on the test environment revealed that it's a directory, unlike the one in our container. This is due to different Linux versions: the host runs CentOS, while the container runs Debian. It was interesting, because the Java 8 base image used Alpine. _Why's the change?_

Unfortunately, [JDK 10 is not yet supported on Alpine](https://github.com/anapsix/docker-alpine-java/issues/69), hence the change of Linux distribution (and also hence the timezone discovery difference, but let's not rush forward too much). As I also found out, not just the one I was working on, but also **our other Java 10 services ware affected as well**: something really smelled around Java 10 and Debian.

## Timezone injection  
The differences between Linux versions started to make clear that _simply mounting resources would never work_. Also, modifying the Dockerfile was a no-go too, because it is processed on the _CI build server_, which uses a different timezone as the target environment.

The next idea was to somehow inject the host's timezone into the container. Docker supports [environment variables](https://docs.docker.com/edge/engine/reference/run/#env-environment-variables) that can be set during container startup, which may be the best possible way to achieve this.

To be able to set the environment variable, I needed the timezone in a [Java consumable format](https://www.mkyong.com/java8/java-display-all-zoneid-and-its-utc-offset/). The Linux command for printing the timezone is `date +%Z%z`, however, several timezones, such as CDT, are not printed in the right format (it should be e.g. CST6CDT).

```bash
$ date +%Z%z
> CDT
```

With some search, I finally found a script for this among [the OSX Docker github issues](https://github.com/docker/for-mac/issues/2396#issuecomment-358150983)! In all Linux distributions, `/etc/localtime` is only a [symlink](https://en.wikipedia.org/wiki/Symbolic_link) pointing to the right directory under `/usr/share/zoneinfo`. Therefore, if I can resolve its path and cut the beginning, I get a well-formed timezone name. My Linux-friendly script for this was:
```bash
$ ls -la /etc/localtime | cut -d/ -f7-
> CST6CDT
```

I then injected it in the runtime configuration by adding `-e TZ=$(ls -la /etc/localtime | cut -d/ -f7-)`. **Of course it didn't work**, but can you guess the value of the environment variable after adding this script to configuration?

```bash
$ echo $TZ
> ls -la /etc/localtime | cut -d/ -f7-
```

<div style="text-align:center"><img src="https://media1.giphy.com/media/xT0xeJpnrWC4XWblEk/giphy.gif"></div>

## Thinking outside of the container  
After gathering the pieces of my blown mind, I managed to find a piece of a mailing list describing [the algorithm of how Java detects timezone on Linux](http://mail.openjdk.java.net/pipermail/core-libs-dev/2013-August/020229.html). It goes like this:

> * If `TZ` environment variable is set, use that
> * If `/etc/timezone` is readable, read the time zone from there
> * If `/etc/localtime` is a symlink, resolve that, and use the name to guess the time zone
> * Otherwise, scan `/usr/share/zoneinfo` for a file whose contents match the contents of `/etc/localtime`

I could never set the `TZ` local variable, despite of the efforts with timezone injection, so the main problem was what I thought before: `/etc/timezone` was readable, and it contained Etc/UTC (which must be the default on Debian), therefore the algorithm didn't even try to do anything with `/etc/localtime` -- mounting it is just not enough. Not to mention, I had no chance to modify the content of `/etc/timezone` or the target of the `/etc/localtime` symlink, simply because I **didn't know the host timezone inside the container** and I wouldn't have liked to **bake it in** either.

The next idea was given by the fourth step of the algorithm. Thanks to mounting, I could _access_ the `/etc/localtime` of the host! This way, I could reach out of the container and read the time settings of the host. Let's see it:

```bash
$ cat /etc/localtime
> CDTCSTCWTCPT
> CST6CDT,M3.2.0,M11.1.0
```

There it is! Now let's use some bash magic!

## Bash magic  
The ~~final working~~ first promising solution was to put this line in the beginning of the app startup script:

```bash
$ export TZ=$(grep -rl '/usr/share/zoneinfo/' -e $(cat /etc/localtime | tail -1) | tail -1 | cut -d/ -f6-)
```

It's so self-explanatary, isn't it? 

<div style="text-align:center"><img src="https://data.whicdn.com/images/43899865/original.gif"></div>

Okay, let's break it down a little bit. As we can see, timezone info is in the last line, so let's tail that:

```bash
$ cat /etc/localtime | tail -1
> CST6CDT,M3.2.0,M11.1.0
```

Now we need to find correct zoneinfo file. The one that contains the same zone as `/etc/localtime`:

```bash
$ grep -rl '/usr/share/zoneinfo/' -e $(cat /etc/localtime | tail -1)
[...]
> /usr/share/zoneinfo/right/America/North_Dakota/Center
> /usr/share/zoneinfo/right/America/North_Dakota/Beulah
> /usr/share/zoneinfo/right/America/Merida
> /usr/share/zoneinfo/right/America/Rankin_Inlet
> /usr/share/zoneinfo/right/CST6CDT
```

We can see that this timezone can be identified by many canonical names. Any of them should work, so let's tail that as well -- it's just personal preference:

```bash
$ grep -rl '/usr/share/zoneinfo/' -e $(cat /etc/localtime | tail -1) | tail -1
> /usr/share/zoneinfo/right/CST6CDT
``` 

This outputs the whole filename, but we only need the timezone's name, so let's cut it:

```bash
$ grep -rl '/usr/share/zoneinfo/' -e $(cat /etc/localtime | tail -1) | tail -1 | cut -d/ -f6-
> CST6CDT
```

Now that we have the correct timezone, let's assign it to the `TZ` environment variable, as timezone detection starts with that. This way we can outwin `/etc/timezone` and the container finally syncs the timezone with the host.

## Or not...
The previous script was working fine for about a month and for many services. But one evening before heading home I got the horrible news: _the logs are in the future on one of the instances_!

![](https://s-media-cache-ak0.pinimg.com/originals/aa/cc/85/aacc857ff1748add531d5b0056d1914e.jpg)

Because now that was a production issue, I spent some time in the evening trying to figure out what goes wrong. I found out that the problem was most probably caused by the nature of `grep` (it seems to be a little indeterministic) and the [tangled zoneinfo folder](https://mm.icann.org/pipermail/tz/2015-February/022024.html) in Linux.

There was an experiment to include leap seconds in the Linux timestamps, but it has never worked correctly so it's not widely used. In some Linux distributions there is a subfolder called 'right' under the `/usr/share/zoneinfo` folder, while others have it as a sibling of this folder. Debian seems to be using the subfolder approach, so `grep` picks up zoneinfo files under 'right' along the ones being directly in zoneinfo. It makes `cut` fail as it uses a fixed number of partitions.

Knowing this, I created this enhanced timezone fix script:

```bash
for f in $(find $zoneinfo -not -wholename $zoneinfo -not -wholename *localtime* -not -wholename *right* -not -wholename *Etc/UTC*); do
  dumped=$(zdump $f)
  currentTime=$(echo $(zdump $localtime) | cut -d" " -f2-)
  zoneTime=$(echo $dumped | cut -d" " -f2-)
  if [[ "$currentTime" == "$zoneTime" ]]; then
    tzfile=$(echo $dumped | cut -d" " -f1)
    break
  fi
done
export TZ=$(echo $tzfile | cut -d/ -f5-)
```
It uses the same algorithm described before: it scans through the hierarchy  under the `/usr/share/zoneinfo` folder looking for a file containing the same time as in `/etc/localtime` -- which is mounted from the host. Now I use `zdump` as it's more straightforward and accurate than tail. During the search the following are excluded:

 * `/usr/share/zoneinfo` and `/etc/localtime` as they do contain the current time, but doesn't code the timezone;
 * `right`: using `zdump` assures that the exact time is compared, so because of leap seconds they would never match. So excluding this is half for optimization and half for making sure the last `cut` will work;
 * `Etc/UTC`: this is a special one, it would always match, so it must be skipped.
 
I even wrote some unit tests in cram as well, but I wouldn't like to go into much details now. If you're interested, I used some ideas from [this article]((https://pbrisbin.com/posts/mocking_bash/)); it's worth checking out.

Testing the enhanced script on the test environment has shown some good results, but on production we weren't so lucky. The script definitely ran on startup and detected the timezone but only in some cases got it right; other times it detected some UTC variant (e.g. Zulu).
 
## Cutting the Gordian knot

In the meantime other teams started upgrading to Java 10 and ran into this timezone issue. One of my colleagues then pointed out that we shouldn't couple the Docker container and the host so much: the app shouldn't be affected by the timezone differences. In fact, only log indexing suffered from this, so it might be possible to solve the problem on Splunk level.

Following his advice, after modifying the timestamp format to `yyyy-MM-dd’T’HH:mm:ss,SSSX`, Splunk was able to calculate the time difference, and the logs didn't appear to be in the future anymore. Well, _only most of the time..._ There were a few log entries that still appeared in the future.

It was because we are using a custom log entry pattern that didn't match anymore due to the changed timestamp format. It meant that for our Java 10 upgraded services, Splunk used a fallback -- its default configuration. The good news is that the default configuration takes timezones into consideration, but [looks for a matching timestamp only among the first 128 characters of the entry](https://docs.splunk.com/Documentation/Splunk/latest/Admin/Propsconf?utm_source=answers&utm_medium=in-answer&utm_term=props.conf&utm_campaign=refdoc#Timestamp_extraction_configuration). I don't know about you, but my team loves using long classnames and deep package structures, that sometimes exceed this character limit, pushing the timestamp outside of the range. Placing the timestamps in the beginning of the entries finally solved the issue for good.

<div style="text-align:center"><img src="https://i.giphy.com/media/oT0a4Cr3P3xRK/giphy.gif"></div>
