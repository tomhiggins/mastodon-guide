# Running a Mastodon instance on Ubuntu 16.04 with Docker & nginx

This guide assumes you have:

1. Root access to a fresh Ubuntu 16.04 x64 machine.
2. A domain name (preferably a wacky TLD) pointed towards the IP address of the machine
3. Some patience... please work all the way through to the "Troubleshooting" section before calling it a day 😃

These instructions have been tested and verified with the standard Ubuntu 16.04 x64 image on both a $10/mo [Vultr](https://vultr.com) instance and a $20/mo [DigitalOcean](https://digitalocean.com) instance. 2GB memory is recommended. You'll also find things faster if you can select the region nearest you geographically. 

## Step 1. Create non-root user with root privileges

First login via SSH as the root user, then add a new user:

`adduser mastodon`

Then give the user root privileges:

`usermod -aG sudo mastodon`

Switch to the new user:

`su - mastodon`

***If you'd like to setup public key authentication, now is the time to do so... it's recommended and Google is your friend for finding an easy guide. Alternatively, you can go right ahead with simple username/password authentication if that's your bag.***

## Install Docker

First, update the package database:

`sudo apt-get update`

Now let's install Docker. Add the GPG key for the official Docker repository to the system:

`sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D`

Add the Docker repository to APT sources:

`sudo apt-add-repository 'deb https://apt.dockerproject.org/repo ubuntu-xenial main'`

Update the package database with the Docker packages from the newly added repo:

`sudo apt-get update`

Tell the system to install from the Docker repo instead of the default Ubuntu 16.04 repo:

`apt-cache policy docker-engine`

Finally, install Docker:

`sudo apt-get install -y docker-engine`

Docker should now be installed, the daemon started, and the process enabled to start on boot. Check that it's running:

`sudo systemctl status docker`

Look for the text "active (running)" under the docker.service

So you don't have to prepend `sudo` to every `docker` command in the future, add yourself to the `docker` group that was just created:

`sudo usermod -aG docker $(whoami)`

Exit the SSH session and then log back in before moving to the next step.

## Install Docker Compose

Check the latest release number [here](https://github.com/docker/compose/releases) and run the following command, swapping out the release number ("1.12.0" below): 

`sudo curl -o /usr/local/bin/docker-compose -L "https://github.com/docker/compose/releases/download/1.12.0/docker-compose-$(uname -s)-$(uname -m)"`

Set the necessary permissions:

`sudo chmod +x /usr/local/bin/docker-compose`

Verify it installed by checking the version:

`docker-compose -v`

If you see the version number, we're in business. 

## Install Mastodon

We're now reading to clone the git repo and start setting up Mastodon. Make sure you're in your home directory by running:

`cd /home/mastodon`

Clone the repo into this directory:

`git clone https://github.com/tootsuite/mastodon.git`

Next up, navigate to the cloned app directory:

`cd mastodon`

Copy the production environment configuration file:

`cp .env.production.sample .env.production`

We need to generate some secret keys before we configure everything, so we'll build the Docker image and get it running unconfigured first:

`docker-compose build` ***<== this may take a while, go make yourself a cup of tea*** ☕️

Now that the image is built, we're going to generate 3 secret keys that are required for the configuration. Run this command 3 times and note down each of the keys generated, we'll need them in a minute:

`docker-compose run --rm web rake secret`

Once you have your 3 secret keys noted somewhere, it's time to edit the configuration file:

`nano .env.production`

Edit the settings, and use the 3 secret keys you generated to populate the `PAPERCLIP_SECRET`, `SECRET_KEY_BASE` and `OTP_SECRET` settings. The order in which you copy/paste the keys doesn't matter, so long as each value is different. 

Next, we need to update the following values:

- **(Required)** Change the `LOCAL_DOMAIN` setting to match the domain you have pointing at the machine.
- **(Required)** Email settings:
	- [Setup a Mailgun account](https://app.mailgun.com/new/signup/) (make sure to enter your CC details for the 10k/mo free email tier)
	- Go through the domain verification process and make sure the domain is listed as green and "active"
	- From the [domains page](https://app.mailgun.com/app/domains) click your domain and then copy/paste the "Default SMTP Login" into your config file as the `SMTP_LOGIN` value and the "Default Password" as the `SMTP_PASSWORD`
	- Change the `SMTP_FROM_ADDRESS` value to something like notifications@<your domain>
- **Optional** If you want to store images and media on S3 instead of your machine (ie. if you don't have a lot of disk space) then generate the required settings via AWS and populate the S3 details (recommended for production use)

Once done, press ^X, type `Y` and hit Enter to save your changes. You can also use any text editor you'd like, if you're not a fan of `nano`

As we've changed the config, we'll need to build again:

`docker-compose build`

Once completed, run the container:

`docker-compose up -d`

Now it's time to run migrations: 

`docker-compose run --rm web rails db:migrate`

We can pre-compile assets too to make things snappier:

`docker-compose run --rm web rails assets:precompile`

One final build as we pre-compiled:

`docker-compose build`

## Setup nginx in front of Docker

First, let's install nginx:

`sudo apt-get install nginx`

We'll remove the default site profile:

`sudo rm /etc/nginx/sites-available/default`

...and it's symbolic link:

`sudo rm /etc/nginx/sites-enabled/default`

Then create a new profile for our Mastodon instance:

`sudo touch /etc/nginx/sites-available/mastodon`

...and a symbolic link enabling it:

`sudo ln -s /etc/nginx/sites-available/mastodon /etc/nginx/sites-enabled/mastodon`

Now let's configure nginx:

`sudo nano /etc/nginx/sites-available/mastodon`

Copy the nginx config from [here](https://github.com/tootsuite/mastodon/blob/master/docs/Running-Mastodon/Production-guide.md) and paste it into the `nano` editor window. 

Find all intances of `example.com` and replace with the domain name you have pointed to your machine, *including* the `ssl_certificate` lines. These are the only changes you should make.

After you've tweaked the file, add this block to the bottom of the file - also adjusting the instances of `example.com`:

```
server {
  listen 80;
  server_name example.com;
  rewrite ^ https://$server_name$request_uri? permanent;
}
```

This will ensure all non-HTTPs traffic is redirected to HTTPS. 

Once done, press ^X, type `Y` and hit Enter to save your changes.

## Setup SSL/HTTPS with Let's Encrypt

First, stop the nginx service:

`sudo systemctl stop nginx.service`

Now we can install Let's Encrypt:

`sudo apt-get install letsencrypt`

Once installed, we can generate the SSL certificates. Run the following command replacing `example.com` with the domain you have pointed at your machine:

`letsencrypt certonly --standalone -d example.com`

Follow the prompts to complete the process. 

## Run Mastodon

We've made a lot of progress, let's get everything running. First, make sure you're in the Mastodon install directory:

`cd /home/mastodon/mastodon`

Now let's stop Docker and rebuild everything to be safe. Run these commands **one line at a time**, making sure none fail:

```
docker-compose down
docker-compose stop
docker-compose build
docker-compose run --rm web rails db:migrate
docker-compose run --rm web rails assets:precompile
docker-compose build
docker-compose up -d
```

Lastly, let's restart nginx:

`sudo systemctl restart nginx.service`

You should be able to visit your domain now, be auto-directed to the HTTPS protocol and see your running Mastodon instance!

## Setup cron jobs to keep feeds working

To keep everything working, we need to set up some cron jobs to regularly tidy things up and refresh feeds. First, make sure you're in the home directory of your `mastodon` user:

`cd /home/mastodon`

Next up, we'll create a file with the commands we'd like to schedule:

`nano mastodon_cron`

Copy/paste in everything below:

```
docker-compose run --rm web rake mastodon:media:clear
docker-compose run --rm web rake mastodon:push:refresh
docker-compose run --rm web rake mastodon:push:clear
docker-compose run --rm web rake mastodon:feeds:clear
```

Once done, press ^X, type `Y` and hit Enter to save your changes. Now we need to tell cron to run this periodically:

`sudo chmod +x mastodon_cron && sudo crontab -e`

If you're prompted to select a text editor, just hit `2` then Enter to select `nano`. The crontab file should open up in edit mode, copy/paste and add this to the end of the file:

`0 0 * * * /home/mastodon/mastodon_cron > /home/mastodon/mastodon_log`

Once again, press ^X, type `Y` and hit Enter to save your changes. This has just told cron to run the specified commands at 12 midnight each day. You can adjust the `/home/mastodon/mastodon_cron` file to change what gets run daily, or change the scheduling by editing the crontab file and saving it again.

## Managing your instance

Information on managing your instance can be found [here](https://github.com/tootsuite/mastodon/blob/master/docs/Running-Mastodon/Administration-guide.md) on the Mastodon repository itself. Before you start editing site settings etc. however, you need to make yourself an admin. 

### Granting admin permissions

After registering on your newly created instance and confirming your account, you'll need to navigate to your Mastodon directory:

`cd /home/mastodon/mastodon`

Then run this command:

`docker-compose run --rm web rails mastodon:make_admin USERNAME=yourusername`

Logout and log back in again, then visit your.domain/admin/settings to start customizing your instance.

## Coming Soon: Converting your instance to Single User

Check back later for more info on this! 👍

## Troubleshooting

### The site loads but looks funky / doesn't respond properly

If you are seeing server errors or assets not rendering on the frontend, something might just need a gentle push to rebuild and start working. 

A good first step is to run this block of commands **one line at a time** from your `/home/mastodon/mastodon` directory, making sure none of them fail - then check https://your.domain again to see if things are working.

```
docker-compose down
docker-compose stop
docker-compose build
docker-compose run --rm web rails db:migrate
docker-compose run --rm web rails assets:precompile
docker-compose build
docker-compose up -d
sudo systemctl restart nginx.service
```

### Email confirmation isn't working

Some mail hosts will blacklist or bounce emails coming from newly created email addresses or domains. Thankfully Mailgun will try sending these messages again and it should eventually get through. 

If you are not receiving emails and need to confirm a user account, go to the "Logs" tab inside the Mailgun UI and click your domain. You should see a green row with the verification email that it attempted to send. Click the cog icon to the left of the row and then "View Message", from here you can manually copy/paste the confirmation URL into your browser to get around this problem.

After your instance is live for ~24 hours, you shouldn't be having this issue any longer. 
