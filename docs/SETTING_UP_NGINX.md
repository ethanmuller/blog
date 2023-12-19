# Setting up a website on Linux in 2024

Google and Apple control the Play Store and the App Store respectively. To
publish an app, you have to get Google's or Apple's permission. You also need
to pay them money for the privilege of adding content to their storefronts. You
also need to pay them a 30% commission on top of that. They own the
storefronts, so they make the rules.

Websites are different. There is no single organization that hosts The Web, so
you can publish a website without anybody's permission.  I find this
empowering. Web development is a much quicker way to get creations in front of
people. **You can't publish a new app today, but you can publish a new website
today.**

Here's an overview on how to set up a website on Linux. Note that you don't
have to be using Linux on your own computer&mdash;it's only needed on the
server.

You don't have to follow along step-by-step; this is meant to be more of an
introduction to these technologies so you understand how they fit together. <small>It's also a reference for myself for when I inevitably forget how to do this.</small>

## This is the hard way

There are much easier ways to throw a website up onto The Web. [Vercel](https://vercel.com/) and [Netlify](https://www.netlify.com/) are two popular choices. However, I'd rather show you how to do it from scratch.

It's worth learning how to host websites the hard way because the skills you learn along the way are generalized, and the end result is much more flexible.

## Servers and IP addresses and domain names

How do I make a website visible on the web? If I have HTML, what do I need to do with it so that anyone in the world can
access it by typing an address into their browser? I need two things:

1. **A computer that can serve my website** (`sudo` access required)
2. **A domain name**

**For my computer:** I pay for a <abbr title="virtual private
server">VPS</abbr> to access a server that is always running. I use `ssh` to
log into it, which is just like opening a terminal window, except it can be
done remotely. This VPS is the computer I serve my website from.

**For my domain name:** I also bought the domain name `mush.network` in order to
point it at my server's public IP address: `172.234.84.182`. To do this, I
logged into my domain name provider and made a DNS record with a type of `A`
and a value of `172.234.84.182`.

After waiting a moment for this change to propagate, I can confirm my domain
name points to my IP address by running the following command:

```shell
ping mush.network
```

A successful ping to a domain name will return a pong from the IP address it points to:

```
PING mush.network (172.234.84.182) 56(84) bytes of data.
64 bytes from vps-jp (172.234.84.182): icmp_seq=1 ttl=47 time=165 ms
64 bytes from vps-jp (172.234.84.182): icmp_seq=2 ttl=47 time=164 ms
64 bytes from vps-jp (172.234.84.182): icmp_seq=3 ttl=47 time=164 ms
64 bytes from vps-jp (172.234.84.182): icmp_seq=4 ttl=47 time=165 ms
```

Cool. This proves the domain name is pointing to the right place.

At this point, if we attempt to access this domain name in the browser, we'll get nothing. We have a server but we need a *web* server.

## NGINX is a web server

NGINX is a rock-solid, very fast web server. Faster than Node.
It's very good at serving static files without any server-side processing.

If your server is running multiple Node apps on different ports,
NGINX can be used to direct incoming web traffic to these Node apps.
This concept is called a *reverse proxy*.

We're also going to use NGINX to handle encryption through HTTPS.

### Why use HTTPS?

If domain names are the usernames of the open web, then HTTPS certificates are
the blue checkmark of the open web. If your browser detects valid certificates,
it can safely declare that its connection to the server on a given domain is
encrypted.

Imagine a world without HTTPS: an evil motel owner could redirect all the
gmail.com requests on his network to his own evil clone of gmail.com.
HTTPS prevents this scenario by disallowing traffic when there's a mismatch
between a domain name and the identity of the server it's paired with. It
guarantees that your traffic isn't tampered with. For example, it shuts down
the possibility of your ISP sneaking in advertisements by inserting HTML into
the pages you're requesting.

Most non-HTTPS websites are non-malicious, but you should still never
ever send login credentials or sensitive information through a non-HTTPS
website, since you have no guarantee that nobody's reading your traffic. So you
shouldn't <em>trust</em> an HTTP website, but you don't need to trust a website
to view it.

## Installing NGINX 

Download NGINX from your distro's package manager. I'm on Arch Linux, so I do
this with:

```shell
sudo pacman -S nginx
```

## Start NGINX whenever I reboot

On systemd-based distros (Ubuntu, Arch, Fedora), the following command will
start the NGINX server immediately, and will restart whenever rebooted:

```shell
sudo systemctl enable --now nginx
```

As long as this didn't give you error messages, NGINX should now be up and
running on port 80. That means if you type your server's public IP address into
your browser, you should now see NGINX's default page:

> **Welcome to nginx!** If you see this page, the nginx web server is successfully installed and working. Further configuration is required.

**We're in.** This means our browser can successfully get responses from NGINX. This is a cause for celebration.

## HTTPS in NGINX

I'm going to use certbot to automatically issue certificates through the
certificate authority called Let's Encrypt. The final command in this section
will programmatically edit our NGINX config, so it's not a bad idea to make a
copy of this file before running these commands.

First of all, we need to tell NGINX what the domain name of our server is. I'll
add the following line just inside the `server` block in my NGINX config:

```nginx
server_name mush.network;
```

And reload this change in NGINX
 
```shell
sudo systemctl reload nginx
```

Next, I install certbot:

```shell
sudo pacman -S certbot certbot-nginx
```

And finally I run certbot:

```shell
sudo certbot
```

This script might catch errors you've made while configuring, so pay close
attention to the output.

But if this command runs without errors, certificates are now set up. At this
point, typing "mush.network" into my browser will direct me to the same default
page as before, but this time through HTTPS, displaying that sweet sweet locked
padlock next to the URL.

If you made it to this point, this is cause for another celebration.

## Serve a directory of files

NGINX has a feature to allow users to browse through arbitrary files nested
within a given directory. It's very barebones, but it can be a great solution
if you just need to distribute files, and don't want to bother updating HTML
every time you add a file.

I want to make this file browser available at [mush.network/files](https://mush.network/files).
To enable this, I edit my NGINX config to include this block within the `server` block:

```nginx
location /files {
    alias /home/me/public;
    autoindex on;
}
```

The value for `alias` is the directory we want to make browseable; in this case,
the `public` directory within the home directory of the user named `me`. The
`autoindex` option is what enables the file listing.

**All the directories nested within will be made public** so don't point this at any directories containing sensitive info like API keys.

Again, you'll have to manually reload any NGINX config changes:
 
```shell
sudo systemctl reload nginx
```

## Make a cool index page

I want to have a custom homepage that directs people to browse through files at the `/files` route we added previously.

By default, putting a file called `index.html` into a directory served by NGINX will serve that file when accessing the directory's path. If I were to put `index.html` directly into the `public` directory, it would override the autoindex page. For that reason, I'll put it into the `public/homepage` directory and use it as my NGINX root for requests to `/`.

Here's a skeleton HTML page to put in `/home/me/public/homepage/index.html`:

```html
<!DOCTYPE html>
<html lang="en-US"><head>
   <head>
       <title>index page</title>

       <!-- this prevents a dorky zoomed-out page on mobile -->
       <meta name="viewport" content="width=device-width,initial-scale=1.0">

       <!-- this tells the browser which characters we're using -->
       <meta charset="UTF-8"> 
   </head>

  <body>
    <p>I'm da index page baby</p>
    <p><a href="/files">browse files</a></p>
  </body>
</html>
```

At this point, since we are serving the whole `public` directory to the `/files` route we should be able to see
this page by visiting [mush.network/files/homepage](https://mush.network/files/homepage).

Now here's the chunk of NGINX config to use actually use this file as the home page. Insert this before before the `/files` block we added previously:

```nginx
location / {
    root   /home/me/public/homepage;
}
```

As always, you gotta reload your NGINX config:

```shell
sudo systemctl reload nginx
```

Now this page is visible directly from top-level [mush.network](https://mush.network).

<footer>

## Further reading

- [Internet Map](https://upload.wikimedia.org/wikipedia/commons/d/d2/Internet_map_1024.jpg)
- [I built a PWA and published it in 3 app stores. Here's what I learned.](https://debuggerdotbreak.judahgabriel.com/2018/04/13/i-built-a-pwa-and-published-it-in-3-app-stores-heres-what-i-learned/)
- [Better explanation of the role of HTTPS](https://blog.mozilla.org/en/products/firefox/https-protect/)
- [NGINX Beginner's Guide](https://nginx.org/en/docs/beginners_guide.html)
- [Command Line Crash Course](https://developer.mozilla.org/en-US/docs/Learn/Tools_and_testing/Understanding_client-side_tools/Command_line)
- [Arch Linux Wiki](https://wiki.archlinux.org/)
- [NGINX + Socket.IO](https://www.nginx.com/blog/nginx-nodejs-websockets-socketio/)

</footer>
