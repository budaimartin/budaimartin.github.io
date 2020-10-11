---
layout: post
title:  "Setting up a CS:GO server in the AWS cloud"
date:   2020-10-12 10:00:00 +0200
categories: Blog
---

During the COVID-19 global pandemic, my company decided to force every employee to work from home until further notice. In the beginning, it was a dream come true: no more commuting to work, saving time, money and frustration. However, as days passed by, separation started to have an impact on team morale. To mitigate this, we started having morning coffee talks over Skype and such, but those always faded into doing something work-related -- we needed something pure fun. Like playing CS:GO in the evenings!

## The need for a server

This idea worked, we have at least one session per week nowadays, even in the 7th month of the work-from-home era. The needs were growing rapidly as more and more current and ex members joined the nightly games, reaching some limitations.

 * Ideally, we want to play in one team, but without a dedicated server, a team can have up to 5 players only.
 * There is no possibility to disable auto-balance. It's fun having no bots in our team and having lots of them in the other!
 * Sometimes it's fun to have a full-house game, with a custom server we can play 10 vs. 10 games with lots of bots running around.
 * We can control all sorts of things in the server like bot difficulty, map groups, change map on demand and so on.

First, we purchased a server on a host provider, which made all of these possible -- at a cost. _Is there a way to create one for free?_ Well, maybe.

<div style="text-align:center"><img src="/img/blog-cs-go-in-cloud/answer-that-for-money.gif"></div>

## Why AWS?

At this time I was studying for my AWS Developer Associate certification, so my first idea was to use the AWS cloud as long as the free tier is enough. For our use case this includes (per month):

 * 750 hours of EC2 t2.micro instance usage -- this is basically one instance running all time
 * 30 GB of general purpose Elastic Block Storage
 * 15 GB of outbound bandwith

## Dockerized CS:GO server

I'm just facinated by Docker, so my first thought was to find an appropriate base image and use that. This way, I can do experiments on my local machine then simply launch it on an EC2 instance when I'm done (unfortunately, by the time writing this, there is no other free Docker-base options available).

I found [Gonzih's docker image](https://github.com/Gonzih/docker-csgo-server), however I ran into several issues. The biggest painpoint was that it contains an outdated version, so an update is needed each time I build the image, increasing it's side enourmosly (above 40GB), which would fall out of AWS free tier.

If this doesn't bother you, or just want to play around with it locally, I would definitely suggest you trying it out.

## Installing the server

So, let's get to it!

### Launching the instance

 1. First, we need to [create an AWS account](https://portal.aws.amazon.com/billing/signup#/start). I think it's pretty straightforward -- only thing is, that you must provide a credit card number, although you won't be charged until you reach the end of the free tier; I will show you at the end of this article how to set an alert for that.
 2. Then sign in (as a root user) and select the AWS region closest to you. Having located in Hungary, I chose eu-central-1 (Frankfurt). This step is important, as you can reduce latency to your server, which is crucial for online gaming.
 3. Navigate to services and select EC2, then launch an instance.
 4. I chose Ubuntu Server 20.04 LTS, because that's what I'm most familiar with, and we won't use anything that would be preinstalled on Amazon Linux -- they would just take up space unnecessarily.
 5. Instance details:
     * Feel free to use the default VPC and set no preference for subnet
     * Auto-assign a public IP, however we will replace this with Elactic IP later
     * Everything else can be left on default settings
 6. Add 30 GB storate. This is the largest you can set for the free tier, and you will need it as the server takes up more than 20 GB.
 7. Add the "Name" tag so you will be able to identify this instance more easily.
 8. Create a Security Group allowing all outbound traffic and inbound traffic for SSH and the server from all sources (0.0.0.0/0)
     * TCP port 22
     * UPD and TCP port 27015
 9. Create and download a new key-pair. Keep this safe, because this is the only way to SSH into your running instance.
 10. I suggest creating an Elastic IP and assigning to your instance. As long as it is attached to a **running** instance, it's free, and you can re-assign it to another instance later if you have to, without having server users update their local configuration with a new IP.
     * On the EC2 dashboard, select Elactic IP
     * Select Allocate Elactic IP address
     * Select it and assign it to your running instance
 
### Configuring the server

 1. Open up an SSH client and connect to your launched instance. You can use PuTTY or git bash on Windows, or the built-in ssh client for other operating systems.
 ```
 ssh -i your_key_pair.pem ubuntu@client_ip
 ```
 2. In case you encountered with _“UNPROTECTED PRIVATE KEY FILE!”_ error, run the following command and try again.
 ```
 sudo chmod 600 your_key_pair.pem
 ```
 3. Create a folder for your server files. The location doesn't matter, however, I don't use the current users's home directory, as later I'd like to control the server with a user with less privledges.
 ```
 sudo mkdir /var/csgosv
 ```
 4. Install updates and add repository for the next steps.
 ```
 sudo -- sh -c 'dpkg --add-architecture i386; add-apt-repository multiverse; apt-get update; apt-get -y dist-upgrade'
 ```
 5. Install the Steam CLI
 ```
 sudo apt-get install -y steamcmd
 ```
 6. Start Steam CLI and run installation steps. You can use anonymous login at this step. App ID 740 refers to CS:GO. Feel free to make yourself a coffee at this point, as this will take several minutes.
 ```
 steamcmd
 force_install_dir /var/csgosv
 login anonymous
 app_update 740 validate
 ```
 7. Create a Game Server Login Token in Steam. You'll need this to start your server so that others can connect to it. Follow [this link](https://steamcommunity.com/dev/managegameservers). As you can see, there are several preconditions to do this, so if you have a limited account, this step won't be free. Best workaround is to transfer some money to your steam wallet, which can be used later.

So, at this point you have everything to start your server. In the next sections I will describe some further actions I made to my server.
 
### Start script

I created a script that spins up the CS:GO server. I start it as a screen session, so the server can keep running in the background and it is possible to reattach to it from another SSH session.

  1. SSH into the running instance.
 ```
 ssh -i your_key_pair.pem ubuntu@client_ip
 ```
 2. Install screen.
 ```
 sudo apt-get install screen
 ```
 3. Create this `start.sh` script in your CS:GO server root directory (which is `/var/csgosv` if you followed me word-by-word).
 ```
 #!/bin/sh
 screen -dmS csgo bash -c './srcds_run -game csgo -console -usercon +map office +hostname YOUR_HOST_NAME +sv_setsteamaccount YOUR_GAME_SERVER_LOGIN_TOKEN +sv_password PASSWORD_FOR_SERVER'
 screen -x
 ```
 First line starts a screen session named _csgo_ and launches a command that spins up the server starting on map _Office_, with the given host name and password. The Game Server Login token is provided here as well. Last line atteches the console to this screen session.

Running this script will start the server in a way that you will be able to reattach to it anytime. If you close your SSH client, open another session, the `screen -r csgo` will take you back where you left off -- you have full control over your server again.

### Create a user for server handling

To start and control the server, you need to login to it via SSH. I created a separate user for this and allowed password login to the server, so other guys in the team can do that, further raising the [Bus factor](https://en.wikipedia.org/wiki/Bus_factor) of our project.

 1. SSH into the running instance.
 ```
 ssh -i your_key_pair.pem ubuntu@client_ip
 ```
 2. Add a user.
 ```
 sudo adduser user_name
 ```
 3. Enter a password for the user.

This new user won't be on the sudoers list, so after signing in with it, your instance will be more protected.

### Attach to server on SSH login

I created the user only for controlling the server. I'd like it to automatically start the server or attach to the running instance on login. It is very convenient and to be honest, pretty cool as well. I created this script and copied to the `/etc/profile.d` directory.

```
#!/bin/sh
echo " ____    ____     ___     ______      ______    _________   ________   _______     ______       "
echo "|_   \  /   _|  .'   \`.  |_   _ \   .' ____ \  |  _   _  | |_   __  | |_   __ \  .' ____ \      "
echo "  |   \/   |   /  .-.  \   | |_) |  | (___ \_| |_/ | | \_|   | |_ \_|   | |__) | | (___ \_|     "
echo "  | |\  /| |   | |   | |   |  __'.   _.____\`.      | |       |  _| _    |  __ /   _.____\`.      "
echo " _| |_\/_| |_  \  \`-'  /  _| |__) | | \____) |    _| |_     _| |__/ |  _| |  \ \_| \____) |     "
echo "|_____||_____|  \`.___.'  |_______/   \______.'   |_____|   |________| |____| |___|\______.'     "
echo "   ______   ______          ______     ___                                                      "
echo " .' ___  |.' ____ \   _   .' ___  |  .'   \`.                                                    "
echo "/ .'   \_|| (___ \_| (_) / .'   \_| /  .-.  \  .--.  .---.  _ .--.  _   __  .---.  _ .--.       "
echo "| |        _.____\`.   _  | |   ____ | |   | | ( (\`\]/ /__\\[ \`/'\`\][ \ [  ]/ /__\\[ \`/'\`\]      "
echo "\ \`.___.'\| \____) | (_) \ \`.___]  |\  \`-'  /  \`'.'.| \__., | |     \ \/ / | \__., | |          "
echo " \`.____ .' \______.'      \`._____.'  \`.___.'  [\__) )'.__.'[___]     \__/   '.__.'[___]         "
echo ""

if [ "$(whoami)" = "SERVER_CONTROLLER_USER" ]; then
  if [ -z "$(ps -la | grep srcds_linux )" ]; then
    echo "Server is not running. Restarting..."
    sleep 2
    cd /var/csgosv
    ./start.sh
  else
    echo "Server is running, reattaching session..."
    sleep 2
    screen -r csgo
  fi
else
  echo "Not SERVER_CONTROLLER_USER, skipping server initialization"
fi
```

Of course, SERVER_CONTROLLER_USER is the user we created before. Oh, and the ASCII art is there just for fun, and `sleep 2` so the one who joins can admire it -- don't judge me please.

Finally, to enable password authentication, change the `PasswordAuthentication no` to `yes` in the `/etc/ssh/sshd_config` file. So, if you don't provide the key file, you will be prompted to provide a password.

## Using the server

Let's see what we've done. SSH into the server using our special user and add the server to our favorites. You can then join the server and have fun while controling your server via SSH.

You must be wondering how the server performs. I mean, on one hand, using one of the smallest instances available in the AWS cloud doesn't sound super effective, and on the other hand, 15 GB of free outbound bandwith might not be enough.

If you'd like to use your server 24/7 with many users, then you concerns might be valid. Otherwise, for 4-5 hour long sessions per week, you will stay below the free tier. Just to make sure, you can observe Cloudwatch metrics to monitor your instance and set a Budget to be notified when you reached your free tier limit (you have 62 budet-months for free, meaning you can have two Budgets set for a full month).

To set a Budget, navigate to the Billing service, select Budgets and then Create budget. You can then have an email notification when the predifined percent of your budget is reached.
