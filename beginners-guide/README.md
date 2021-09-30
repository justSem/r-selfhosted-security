# First: What's going on?

Recently posts have been showing up about people finding others' exposed dashboards or even fully unprotected services such as Heimdall, Pihole, Calibre, you name it. People expose it all on the public web, often without even knowing they're doing so.

To some this might seem innocent, but it's not. Even if you're not a specific target to anyone, there a lots of automated bots and botnets out there who just scan the entire internet for exposed services like yours in order to exploit those.

 

**So what are the dangers of this exactly?**

Those services you're hosting are exposing a lot of your private info. I'll list a few examples of things I come across.

* I once came across a fully open Calibre instance, upon browsing through it I found out that this particular person configured Calibres mail settings using their GMail details, just a little tinkering exposed their full GMail username and password
* People tend to use their full names, or even full address info, etc. in things like Nextcloud, maybe even things like Pihole or Heimdall. This *will* make you a target for (automated) phishing campaigns. If those services are publicly accessible you can easily assume that someone has already got his hands on your info.

So this all might seem innocuous to some, or some might even utter the: But I have nothing to hide - kind of phrase. But think about why most people are self-hosting in the first place. Privacy is most likely a big part of that, and now you're putting that out on the web for everyone to see?

In example: Big data, botnets, hackers, etc.  can build an extensive profile based on this kind of info:

* One could sift through your Calibre service to find out what things you read.
* One could sift through your Pihole logs to find out what you do on the web.
* One could search through your Plex, Jellyfin, or others to find out what things you like to watch.

This kind of info is especially useful for things like Phishing campaigns. The more familiar and polished a phishing mail is, the more likely you'll fall for it. And you *will* be targeted. No-one's exempt.

 

Another danger is the case where people have a set-and-forget mentality, which leads them to never updating their services. In that case your service **will** get hacked at some point which might result in anything from your device being abused as cryptominer, to your connection being abused for malicious traffic, your devices being enslaved into a botnet or an actual human hacker who might have even more sinister intents.


Additionally some router models allow to automatically open ports via the UPnP protocol (short for *universal plug and play*). That's very convenient for hardware manufacturers, as they don't need to guide customers through a router interface that's going to be different on every model. This takes away your control over which ports are allowed to be reachable over the internet and you rely on the router manufacturer to make sure your router firmware is patched. Plus you easily might forget that you have activated that setting at some point in the past.


# How do I know if I'm publicly exposing services?

There are a few indicators which will easily tell you:

* Did you ever follow a guide that told you to port-forward something?
* Do you proxy or forward your services using a reverse proxy? (i.e. Nginx proxy manager)
* Can you access your services from anywhere (i.e. from your phone) without any extra effort like a VPN.

 

**I'm not sure, how do I check?**

There are plenty of tools that will freely tell you if you're hosting something. First you'll need to know your public IP. Some site like [https://whatismyipaddress.com/](https://whatismyipaddress.com/) will tell you.

Please realise you might have a number of different IP addresses dependent on if your provider provides you with both IPv4 and/or IPv6. Your *public* IPv4 address will be the same for all devices in your network, but your IPv6 address will be different *per device!*

The following tools might give you an insight in the ports you have opened publicly:

* **Shodan** [https://shodan.io](https://shodan.io) \- Shodan does it's own scanning but will not per-say reveal everything as it does not tend to scan every single open port at any given time. Some IP addresses might not even be listed in Shodan.
* **Yougetsignal** [https://www.yougetsignal.com/tools/open-ports/](https://www.yougetsignal.com/tools/open-ports/)  \- Chances are that if you've been port forwarding you've been using a tool like this to actually verify if the port you've configured is accessible.
* **ShieldsUp!** [https://www.grc.com/x/ne.dll?bh0bkyd2](https://www.grc.com/x/ne.dll?bh0bkyd2) \- Steve Gibson's classic tool will scan common ports, check for exposed Windows services, or scan a custom range of ports that you specify.

**I'm still unsure and I want to scan it all, how do I do that?**

This section is slightly more advanced, but if you can selfhost then you can do this too!

First you'll need a device that does *not* host any of your services and a **different internet connection**. (Your phone's 4G or a neighbours WiFi will do).

You'll need a port scanning tool, in this case I'll use nmap which is available for practically all linux distributions, macOS and Windows.

If you're using Windows you can download nmap here: [https://nmap.org/download.html](https://nmap.org/download.html)

If you're using a Debian based distro (Debian, Ubuntu, Mint, etc.)  you can install nmap using `sudo apt install nmap`

If you're using a Redhat based distro (Redhat, Fedora, CentOS, etc.) you can install nmap using `sudo` `dnf install nmap`

If you're using macOS you can install nmap using Homebrew ( [https://brew.sh](https://brew.sh) ) by issuing `brew install nmap`

One you've got nmap setup, make sure you're using a different internet connection and then issue:

    nmap -v -T4 -sV -A -p 1-65535 my.public.ip.address

This will take a while as it'll scan all available TCP ports. It'll also try to determine what's running on an open port it finds (-sV flag) as well as some additional detection (-A flag)

 

# Okay, so I do got open ports, what do I do?

Firstly, you'll have to close them. It's most likely that you'll do this in your router. If you're unsure then I'd suggest you check the guide that you used to setup your service in order to determine what steps you took to expose it to the internet in the first place.

 

**So now my ports are closed, but I can't access service** ***xyz*** **from remote anymore. What do I do?**

It's understandable you want to access your services from anywhere, but there are more secure methods for this then simply exposing this.

There are a number of steps you can take which'll be listed in order from most secure to least.

* Use a VPN
   * Setting up a VPN like [Wireguard](https://www.wireguard.com/) is easy and secure. WireGuard has support for all major devices and it'll allow you to access your entire network from anywhere.
   * *Sidenote: You'll have to port forward WireGuard from your router, this is to be expected. But exposing a VPN service to the public internet is way more secure then exposing an unsecured service.*
* Use port-forwarding with specific IPs
   * This is a feature some routers might not support. But you can utilize a whitelist of IPs that can access your service.
* Using Cloudflare's[Argo tunnel](https://blog.cloudflare.com/argo-tunnel/)
   * By using Cloudflare's Argo tunnel you don't have to open any ports, but instead your webserver will build up a vpn-like connection to cloudflare, over which your webserver will be reachable to cloudflare. Your users then access your service through cloudflare without any risk for you due to exposed ports.
* Utilizing a security CDN like CloudFlare
   * Using services like CloudFlare prevents an attacker from learning your actual IP address (unless said IP address can be accessed somehow through your service of course). Additionally CloudFlare actively filters out bots and malicious traffic. Depending on your tier with them you have more granular control and can choose to block entire countries from accessing your site.
* Use a reverse proxy with an authentication frontend
   * One could utilize a platform like [Authelia](https://www.authelia.com/) or [Keycloak](https://www.keycloak.org/) to secure public-facing services.
* Use a reverse proxy and utilize access-lists
   * A thing one could do with a reverse proxy like nginx is the usage of access lists. By using the `allow` [directive](https://nginx.org/en/docs/http/ngx_http_access_module.html) in the nginx config you can restrict entire services or subfolders to specific IP addresses.

 

**I've read this all, but I still keep wanting to do the things I do. Any tips?**

* Be aware of what info you expose using the services you expose to the internet.
* CHANGE DEFAULT PASSWORDS! This cannot be said enough, exposing services is one thing, but not changing passwords is like giving out your credit card to complete strangers and hoping they'll bring it back to you.

 

# General recommendations

These might be duplicates of parts above, but it's useful to sum them up:

1. Expose only what's really needed: Why would your service need to be open to the internet?
2. Change default passwords: You don't give your credit card to strangers either, do you?
3. Use common sense: You can't magically access something you host at home without exposing something to the public internet.
4. Use 2FA wherever you can. Any form of 2FA is better then nothing. Most services support OTP (Google Authenticator/Authy/Yubico Auth) these days and the more advanced ones even support Webauthn (Yubikeys or any other hardware token)
5. Make sure UPnP is disabed in your router.
 

To-do parts:

* >!Extend on how-tos in building Wireguard, Nginx and NAT access lists!<



Changelog:

* >!Added Clouflare's Argo Tunnel!<
* >!Added 2FA and Cloudflare; Clarified requirement for separate connection for nmap.!<
* >!Initial guide!<
