# Converting your instance to single user mode

*This guide assumes you have set up an instance following the [Up & Running Guide](https://github.com/ummjackson/mastodon-guide/blob/master/up-and-running.md).*

If you plan on using your instance just for yourself (eg. username@my.domain) or would prefer not to manage an instance with multiple users, then you can convert to "single user mode". This will default the home page to your profile and disable registrations. 

**Important**: Converting to single user mode will use the *first* user created on the instance as the default user. It's recommended that you register only 1 user on the instance before converting over. 

First, while logged in as the `mastodon` user, navigate to the install directory:

`cd /home/mastodon/mastodon`

Next up, let's edit our `.env.production` file:

`nano .env.production`

Scroll down until you see a block that looks like this:

```
# Registrations
# Single user mode will disable registrations and redirect frontpage to the first profile
# SINGLE_USER_MODE=true
```

Edit the third line above and remove the #, so it looks like this:

```
# Registrations
# Single user mode will disable registrations and redirect frontpage to the first profile
SINGLE_USER_MODE=true
```

Once done, press ^X, type Y and hit Enter to save your changes.

Now let's rebuild the updated containers without touching the database:

`docker-compose build`

..and bring it online:

`docker-compose up -d`

Wait up to a minute for nginx to start talking with the container again, then refresh and you should now be in single user mode!

**Note:** If you keep receving a "502 Bad Gateway" error, you can force it along with a restart of nginx:

`sudo systemctl restart nginx.service`
