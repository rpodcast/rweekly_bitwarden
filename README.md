## Introduction

This document summarizes how to create a self-hosted instance of the [Bitwarden](https://bitwarden.com/) password manager service to store R-Weekly account credentials for various third-party services.  While there are a few different methods of self-hosting Bitwarden, I chose the implementation written in Rust using [bitwarden_rs](https://github.com/dani-garcia/bitwarden_rs) based on recommendations given in [episode 316](https://linuxunplugged.com/316) of the [Linux Unplugged](https://linuxunplugged.com) podcast.  

## Host Environment Setup

Any linux virtual server that is capable of running [Docker](https://www.docker.com/) will meet the needs.  For this project, I am using a server deployed on [Digital Ocean](https://digitalocean.com) with the following specifications:

* Linux distribution: Ubuntu 18.04 LTS
* Memory: 1 GB
* Storage space: 25 GB

Here are the steps to prepare the server for running Bitwarden:

1. Create a new server with the specifications above and ensure that you can connect to the server via SSH.  For reference see [this document](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/).
2. Add a non-root user per [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-ubuntu-18-04).  Make a note of the new user UID and GID on the system.  You can find those values with the following commands:

```
# uid
id -u {username}

# gid
id -g {username}
```

3. Install docker per [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04).
4. Install docker-compose per [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-ubuntu-18-04).
5. Map a domain name to the server (specifically an A record).  At this time, I have mapped the domain `bit.r-podcast.org` to the server from my `r-podcast.org` domain service (Digital Ocean). For reference see [this guide](https://www.digitalocean.com/docs/networking/dns/how-to/manage-records).

## Bitwarden setup

Much of this procedure was adapted from information contained in the [bitwarden_rs wiki](https://github.com/dani-garcia/bitwarden_rs/wiki).

1. Connect to the server and create a directory for the project.  For example `/home/{user_id}/bitwarden`
2. Within this directory, create the following sub-directories: `bw-data`, `caddycerts`.
3. Create a `.env` text file with the following environment variable values:

```
# generate token with following command: openssl rand -base64 48
ADMIN_TOKEN=abcdefghijklmnopqrstuvwxyz1234567891234567891234
# substitute below value with your domain
DOMAIN=b.r-podcast.org
# substitute below value with your email
EMAIL=your_email@somewhere.com
```

4. Create `docker-compose.yml` and `Caddyfile` files using the templates [here](https://github.com/dani-garcia/bitwarden_rs/wiki/Using-Docker-Compose).  I made some modifications to the template `docker-compose.yml` to allow the following tasks:

     * Create log files in the `bw-data` directory for key events
     * After creating your account on the BitWarden server, then remove the capability to sign up for the service, since we can invite our RWeekly team to join the custom organization using the BitWarden web interface.
     * Supply the aforementioned domain name and associated email address for the SSL certificate creation by passing them directly as environment variables
     * Supply the uid and gid values obtained during the linux server set-up as environment variables.  This ensures that all files created in `bw-data` are owned by the same linux user and not the root user, which makes backups easier.

## Running Bitwarden

To run the docker containers, use the following:

```
docker-compose up -d
```

Verify that the installation is running by visiting the domain configured in the previous setup.

To stop the containers, use the following:

```
docker-compose stop
```

To destroy the containers, use the following:

```
docker-compose down
```

## RWeekly organization setup

1. Log in to the BitWarden instance and create your BitWarden account.  Be sure to use the same email address as what you have used in the docker setup.
2. In the Organizations sidebar, click New Organization.  Name the organization `RWeekly` and put your email address in the billing email box. Note that since we are self-hosting BitWarden, there will not be any charges.  Then click Submit.
3. You should see the organization admin interface appear. 
4. Click the Manage tab, and under the Manage section click the Collections sidebar.  Add a new collection by clicking the New Collection button and specify a name of `RWeekly Admin`.  This collection will contain all of the password entries associated with RWeekly.

## Adding users to the organization

1. Log in to the BitWarden instance and visit the organization interface for `RWeekly`
2. Click the Manage tab
3. Click the Invite User dialog box to bring up the invite user form. Add the email address of the new user.  For user type, select either manager or admin.  Under access control, select `The user can access only the selected collections`.  Check the box next to `RWeekly Admin` collection.  Then click save.
4. Log out of BitWarden to get the main login page.
5. In the email address box, type the new user's email address. Leave the password box blank, and then click Create Account.
6. In the new form, add the name of the user in the `Your Name` field, and enter a __temporary__ password in the appropriate fields.  Then click submit.
7. You should be taken back to the login form.  Log in with the new user's email address and temporary password.  If login is successful, go ahead and log back out.
8. Log in as your user account, then visit the organization admin interface for `RWeekly`.  Like before, click the Manage tab.
9. In the people section, you should see the name of the new user with a small alert on the right side indicating they accepted the invitation.  Hover over the row with the new user and click the dropdown next to the gear. You should see a menu entry appear with `confirm`.  Click `confirm`.
10. Send an email to the new user giving them the temporary password and instructions for how to log in to BitWarden and how they can change their password.  I used am email template like the following:

```
Subject: BitWarden account for RWeekly

Body:

Hello {Name of User}, here is the tmp password for your BitWarden account:

{tmp_password}

Please log in to {bitwarden_domain} and change this to your desired password :)  You can find the settings by clicking the little avatar in the upper right corner and selecting Settings.  If you have any questions about how it works, don't hesitate to ask!
```

## TODO

Here are various additional items we should implement, many coming from the project's [wiki](https://github.com/dani-garcia/bitwarden_rs/wiki):

* Create automated backups of the data files directory
* Rotate log files
