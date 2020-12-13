---
layout: post
title:  "Automated CS:GO server startup"
date:   2020-12-13 17:00:00 +0100
categories: [Today I Learned]
---

Two months ago, I published a [blogpost about creating a CS:GO server in the AWS cloud](https://budaimartin.github.io/blog/2020/10/12/csgo-server-in-the-cloud.html). Back then, the idea was quite immature, as it required several manual steps, I only mentioned at the end of the post that most of the process could be automated using [user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) feature of EC2. Now that after a major update I couldn't update my server due to lack of space, I decided to give automation a try, so after an update, I just need to spin up a new EC2 instance, wait, reassign the Elastic IP adress, and continue the fun!

## Prerequisites

During development, I ran into several issues. This process could be more automated using [CloudFormation](https://aws.amazon.com/cloudformation/) or [Terraform](https://www.terraform.io), but I think for now it's just enough.

EC2 user data is a single script file, so the [start script](https://budaimartin.github.io/blog/2020/10/12/csgo-server-in-the-cloud.html#start-script) and the [server reattaching script](https://budaimartin.github.io/blog/2020/10/12/csgo-server-in-the-cloud.html#attach-to-server-on-ssh-login) cannot be uploaded as a part of it. Hence, I uploaded those in an S3 bucket and attached an IAM role with the following policy, so they could be downloaded.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:Get*",
                "s3:List*"
            ],
            "Resource": "arn:aws:s3:::MY_BUCKET_NAME/*"
        }
    ]
}
```

In the resources part, `MY_BUCKET_NAME` is the name of the actual bucket, and `/*` means that all objects in the bucket are in scope. Actions `s3:Get*` and `s3:List*` provides read-only access, which is enough for this use case. The [AWS free tier](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&all-free-tier.q=s3&all-free-tier.q_operator=AND) contains 5GB of standard storage, 20,000 get and 2,000 get requests, which is more than enough for us.

## The user data script

### Installing dependencies

The first part is responsible for upgrading our instance and installing all the required dependencies. The lib32gcc1 is needed because we download the bundled steamcmd, so we can run it in a non-interactive way. Also, since AWS S3 is involved, we need to install the AWS CLI.

```
dpkg --add-architecture i386
add-apt-repository multiverse
apt-get update
apt-get -y dist-upgrade
apt-get install -y lib32gcc1
apt-get install -y awscli
```

### Installing the Steam CLI

This part downloads and extracts the Steam CLI. User data scripts run as root, so I need extra steps to download this in ubuntu's home folder and make it its owner. Ubuntu is the pre-created user with which you can SSH in your EC2 instance when you select an Ubuntu image.

```
mkdir -p /home/ubuntu/steamcmd
cd /home/ubuntu/steamcmd
curl http://media.steampowered.com/client/steamcmd_linux.tar.gz | tar -xvz
chown -R ubuntu /home/ubuntu/steamcmd
```

### Setting up the start script

The next part downloads the startup script in the future server folder. Of course, `MY_BUCKET_NAME` is the name of the bucket mentioned above.

```
mkdir /var/csgosv
aws s3 cp s3://MY_BUCKET_NAME/start.sh /var/csgosv
```

### Creating user for server handling

The following snippet creates a user that can be used for SSH-ing in the instance in order to control the server. This is important, because we don't want to control the server with a user with elevated access. To make the script below work, replace `SERVER_HANDLING_USER` with the desired username, and `ENCRYPTED_PASSWORD` with a generated password. For the latter, you can use the `mkpasswd` command from the `whois` package.

This user will be authenticated by username and password instead of key pair, so this setting needs to be enabled and the SSH service needs to be restarted.

```
useradd -m -p ENCRYPTED_PASSWORD -s /bin/bash SERVER_HANDLING_USER
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
service ssh restart
```

### Setting up the server reattaching script

The script for server reattaching after SSH needs to be downloaded from the S3 bucket.

```
aws s3 cp s3://MY_BUCKET_NAME/reattach.sh /etc/profile.d
```

### Setting up the server

We are almost there! Next step is to download the server. I want the owner to be ubuntu, because I don't want to use the root user for anything in the future. The first line runs the command as ubuntu, while the second line reassures that all files in that folder will be owned by ubuntu, so it can modify and run all the files in it.

```
su -c "/home/ubuntu/steamcmd/steamcmd.sh +login anonymous +force_install_dir /var/csgosv +app_update 740 +quit" - ubuntu
chown -R ubuntu /var/csgosv
```

### Launching the server

This step is optional. If you did everything right, after the script ran, you can SSH in with your SERVER_HANDLING_USER and the server will launch. If you wish to automate server startup as well, place this line at the end of your user data script.

```
su -c "/var/csgosv/start.sh" - SERVER_HANDLING_USER
```

## Monitoring and troubleshooting

First, I'd like to highlight that this script takes very long time to run, because it needs to download 27GB of data, so don't worry if the server isn't up immediately. The AWS console doesn't really help either, because instance state will be Running, and Status checks will be green even if the script is still running, or if it has failed. Best way to troubleshoot is to check the logs:

```
ssh -i cs-go-kp.pem ubuntu@YOUR_INSTANCE_IP "cat /var/log/cloud-init-output.log"
```

You can also check the logs while the script is still running, but be careful, because the script uses lots of resources, so you can overwhelm the instance and it might kill your process. I think it's better to be patient, wait around 20 minutes and check the logs after.

