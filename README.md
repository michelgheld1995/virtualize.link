# [Original author: https://github.com/quietsy/advanced-configurations](https://github.com/quietsy/advanced-configurations)
# [Original Website](https://virtualize.link)

# Why this repo?
The original author quietsy seems to have started working with linuxserver.io and removed the "securing swag" page from his website.
The text is now available at: [Securing Swag](https://www.linuxserver.io/blog/securing-swag)
This write up lacks critical security features quietsy mentioned in earlier texts. Why? I don't know.
This readme will just combine his steps for VPS Proxy and securing swag.
So assume that all credit for write up and work goes to quietsy, I will indicate so otherwise. The code has changed so some configurations will be different. At the time of this README.md creation May 16th, 2026, I have added a Clarification for the Internal Apps section of this guide.

---
tags:
  - Containers
---

# VPS Proxy

![VPS](images/vps.png)

This setup allows you to hide your home IP, protect your privacy and protect your home server against DDOS attacks while keeping all of your data at home.

Once it's up and running, exposing a resource through the VPS is **as simple** as adding one line to your home SWAG.

The TLDR version is:

- Create a VPN tunnel between your home and a VPS
- Configure SWAG to proxy traffic through the tunnel
- Configure Fail2ban to block attackers

There are many ways to create this setup with many variations for many different purposes, in my opinion these containers are easy to work with and to maintain, every container in this setup can be used for other purposes **as well** as being used for the proxy without any compromises:

- Home SWAG can be used as a reverse proxy for all of your other Home server containers.
- Home Wireguard Client can be used to route any container through the VPS.
- VPS SWAG can be used as a reverse proxy for all of your other VPS containers.
- VPS Wireguard Server can be used as your private cloud VPN server.

## Requirements

- A working instance of [SWAG](https://github.com/linuxserver/docker-swag) at home
- A working instance of [SWAG](https://github.com/linuxserver/docker-swag) on the VPS

## Initial VPS Wireguard Server Configuration

Configure your VPS Wireguard Server according to the [Wireguard documentation](https://github.com/linuxserver/docker-wireguard).

```YAML
  wireguard:
    image: ghcr.io/linuxserver/wireguard
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - SERVERURL=external.com
      - SERVERPORT=51820
      - PEERS=1
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.13.13.0
      - ALLOWEDIPS=10.13.13.0/24
    volumes:
      - /path/to/appdata/config:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

**Note that `ALLOWEDIPS` is set to only allow access to the Wireguard subnet.**

Once done start the container and validate that `docker logs wireguard` contains no errors.

## Initial Home Wireguard Client Configuration

Configure your Home Wireguard Client according to the [Wireguard documentation](https://github.com/linuxserver/docker-wireguard).

```YAML
  wireguard:
    image: ghcr.io/linuxserver/wireguard
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
    volumes:
      - /path/to/appdata/config:/config
      - /lib/modules:/lib/modules
    restart: unless-stopped
```

Once done start the container and validate that `docker logs wireguard` contains no errors (Ignore the missing wg0.conf message).

## Connecting the Wireguard Client to the Wireguard Server

Copy `peer1.conf` from your VPS Wireguard Server's `config/peer1/` folder to your Home Wireguard Client's `config` folder and rename it to `wg0.conf`.

Edit your Home Wireguard Client's `wg0.conf`, remove the `DNS` line and add the `PersistentKeepalive = 25` line under `Peer`, it should look like this:
```Nginx
[Interface]
Address = 10.13.13.2
PrivateKey = <private-key>
ListenPort = 51820

[Peer]
PublicKey = <public-key>
Endpoint = <domain>:51820
AllowedIPs = 10.13.13.0/24
PersistentKeepalive = 25
```
Save the changes and restart the container on your Home server with `docker restart wireguard`, validate that `docker logs wireguard` contains no errors.

Validate that the tunnel is working by pinging both sides:

- On the Home server run - `docker exec wireguard ping 10.13.13.1`
- On the VPS run - `docker exec wireguard ping 10.13.13.2`

## Configuring the VPS SWAG to Use the Tunnel

Replace the following lines on the VPS SWAG container:

```YAML
    ports:
      - 443:443
      - 80:80
```

With:

```YAML
    network_mode: "service:wireguard"
    depends_on:
      - wireguard
```

Add the ports under the VPS Wireguard Server container:

```YAML
    ports:
      - 80:80
      - 443:443
      - 51820:51820/udp
```

Add the following to the bottom of the VPS SWAG configuration under `config/nginx/site-confs/default`:

```Nginx
server {
    listen 443 ssl;
    server_name *.external.com;
    include /config/nginx/ssl.conf;
    client_max_body_size 0;

    location / {
        include /config/nginx/proxy.conf;
        resolver 127.0.0.11 valid=30s;
        proxy_pass http://10.13.13.2:8080;
    }
}
```

Recreate the VPS Wireguard Server container to apply the changes, then recreate the VPS SWAG container which depends on the tunnel.

## Configuring the Home SWAG to Use the Tunnel

Replace the following lines on the Home SWAG container:

```YAML
    ports:
      - 443:443
      - 80:80
```

With:

```YAML
    network_mode: "service:wireguard"
    depends_on:
      - wireguard
```

Add the ports under the Home Wireguard Client container:

```YAML
    ports:
      - 80:80
      - 443:443
      - 51820:51820/udp
```

Configure the Home SWAG to see the real IP of connections coming from the tunnel by adding the following inside the `http` section in `config/nginx/nginx.conf`:

```Nginx
	set_real_ip_from 10.13.13.1/32;
	real_ip_header X-Forwarded-For;
```

In order to catch all the unused subdomains and redirect to an error page, add `listen 8080 default_server;` to `config/nginx/site-confs/default` under the main server block:

```Nginx
# main server block
server {
	listen 8080 default_server;
	listen 443 ssl http2 default_server;
```

Expose a container through the tunnel by adding `listen 8080;` to it's proxy configuration, for example:

```Nginx
server {
    listen 8080;
    listen 443 ssl;
    server_name heimdall.*;
    include /config/nginx/ssl.conf;
    client_max_body_size 0;

    location / {
        include /config/nginx/proxy.conf;
        resolver 127.0.0.11 valid=30s;
        set $upstream_app heimdall;
        set $upstream_port 443;
        set $upstream_proto https;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;

    }
}
```

Recreate the Home Wireguard Client container to apply the changes, then recreate the Home SWAG container which depends on the tunnel.

Validate that the containers you exposed now work through the tunnel by browsing `https://<container>.external.com/`.

## Traffic Overview

![VPS2](images/vps2.png)

## Fail2ban

Now that everything is working, Fail2ban should ban the right IP of attackers, but they're coming in through the tunnel and iptables isn't blocking them, therefore we will block them through NGINX.

Create a file called `nginx.conf` in your Home SWAG under `config/fail2ban/action.d/` with the following:

```Nginx
[INCLUDES]

[Definition]

actionstart = touch /config/nginx/blocklist.conf
actionstop = 
actioncheck = 
actionban = grep -qxF "deny <ip>;" /config/nginx/blocklist.conf || echo "deny <ip>;" >> /config/nginx/blocklist.conf
actionunban = sed -i '/deny <ip>;/d' /config/nginx/blocklist.conf

[Init]

name = default
```

Edit `config/fail2ban/jail.local` and add `nginx` to the `action` of all the jails, for example:

```Nginx
[authelia]
enabled  = true
filter   = authelia
port     = http,https
logpath  = /authelia/authelia.log
action  = iptables-allports[name=authelia]
          nginx
```

Add the following line into the `http` section in `config/nginx/nginx.conf`:

```Nginx
	include /config/nginx/blocklist.conf;
```

Restart the Home SWAG to apply the changes with `docker restart swag`.

## Notes

### Exposing more containers
Expose more containers by simply adding `listen 8080;` to their proxy configuration on the Home server, for example:

```Nginx
server {
    listen 8080;
    listen 443 ssl;
```

Restart Home SWAG by running `docker restart swag` to apply the changes.

### Restarting order

If you're experiencing problems and you want to restart everything, the correct order is:

- VPS - `docker restart wireguard`
- VPS - `docker restart swag`
- Home - `docker restart wireguard`
- Home - `docker restart swag`

### Authelia / Authentik

If you expose Authelia/Authentik through the tunnel, you need to make a small adjustment for the redirects to work.

The idea is to force https, since traffic through the tunnel is coming over as http but the VPS exposes https.

Edit Authelia/Authentik confs under `config/nginx/`, replace `$scheme` with `https`.

Restart the Home SWAG to apply the changes with `docker restart swag`.

### Exposing a resource only through one domain but not the other

You control what gets exposed where in 2 ways:

- Through the `listen <port>;` setting, 8080 is through the VPS and 443/80 is directly.
- Through the `server_name something.external.com` setting, if you explicitely specify the full address.

If a resource isn't exposed, the default action under the main server block in your Home SWAG will apply.


### Attackers are filling my logs with Access Denied!

If you want attackers to be redirected instead of showing them an error page and avoid them spamming the logs with 403 errors, add the following inside the `http` section in `config/nginx/nginx.conf` on both SWAGs:

```Nginx
    error_page 400 403 404 444 500 502 503 504 http://www.google.com/;
```

Restart the Home SWAG to apply the changes with `docker restart swag`.

# Securing SWAG
[SWAG](https://github.com/linuxserver/docker-swag) - Secure Web Application Gateway (formerly known as linuxserver/letsencrypt) is a full fledged web server and reverse proxy with Nginx, PHP7, Certbot (Let's Encrypt™ client) and Fail2Ban built in. SWAG allows you to expose applications to the internet, doing so comes with a risk and there are security measures that help reduce that risk. This article details how to configure SWAG and enhance it's security.

## Requirements

- A working instance of [SWAG](https://github.com/linuxserver/docker-swag)

## Monitor SWAG
Use monitoring solutions such as [SWAG Dashboard](https://github.com/linuxserver/docker-mods/tree/swag-dashboard) to keep an eye on the traffic going through SWAG and check for suspicious activity such as:

- A lot of hits from a country unrelated to your users
- A lot of requests to a specific page or static file
- Referers that shouldn't refer to your domain
- A lot of hits on status codes that are not 2xx

## Internal Applications

## Clarification (not by quietsy)
The internal applications step only applies if you use port forwarding with your HOME PUBLIC IP. I think.

Internal applications can be proxied through SWAG in order to use `app.mydomain.com` instead of ip:port, and block them externally so only your local network could access them.

Create a file called `nginx/internal.conf` with the following configuration:

```Nginx
allow 192.168.1.0/24; #Replace with your LAN subnet
deny all;
```

Utilize the lan filter in your configuration by adding the following line inside every location block for every application you want to protect.
```
    include /config/nginx/internal.conf;
```

Example:

```Nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name collabora.*;
    include /config/nginx/ssl.conf;
    client_max_body_size 0;

    location / {
        include /config/nginx/internal.conf;
        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        set $upstream_app collabora;
        set $upstream_port 9980;
        set $upstream_proto https;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;
    }
}
```

Repeat the process for all internal applications and for every location block.

One way to securely access internal applications from the internet is through a VPN, for example WireGuard:

[WireGuard Container](https://hub.docker.com/r/linuxserver/wireguard)

[WireGuard on OPNSense](https://blog.linuxserver.io/2019/11/16/setting-up-wireguard-on-opnsense-android/)


## Fail2Ban
Fail2Ban is an intrusion prevention software that protects external applications from brute-force attacks. Attackers that fail to login to your applications a certain number of times will get blocked from accessing all of your applications.
Fail2Ban looks for failed login attempts in log files, counts the failed attempts in a short period, and bans the IP address of the attacker.

Mount the application logs to SWAG's container by adding a volume for the log to the compose yaml:
```
      - /path/to/nextcloud/nextcloud.log:/nextcloud/nextcloud.log:ro
```
If the application has multiple log files with dates, mount the entire folder:
```
      - /path/to/jellyfin/log:/jellyfin:ro
```
Recreate the container with the log mount, then create a file called `nextcloud.local` under `fail2ban/filter.d`:
```nginx
[Definition]
failregex=^.*Login failed: '?.*'? \(Remote IP: '?<ADDR>'?\).*$
          ^.*\"remoteAddr\":\"<ADDR>\".*Trusted domain error.*$
ignoreregex =
```
The configuration file containes a pattern by which failed login attempts are matched. Test the pattern by failing to login to nextcloud and look for the entry corresponding to your failed attempt.
```
{"reqId":"k5j5H7K3eskXt3hCLSc4i","level":2,"time":"2020-10-14T22:56:14+00:00","remoteAddr":"1.2.3.4","user":"--",
"app":"no app in context","method":"POST","url":"/login","message":"Login failed: username (Remote IP: 5.5.5.5)",
"userAgent":"Mozilla/5.0 (Linux; Android 11; Pixel 5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/5.6.7.8 Mobile 
Safari/537.36","version":"19.0.4.2"}
```
Test the pattern in `nextcloud.local` by running the following command on the docker host:
```
docker exec swag fail2ban-regex /nextcloud/nextcloud.log /config/fail2ban/filter.d/nextcloud.local
```
If the pattern works, you will see matches corresponding to the amount of failed login attempts:
```
Lines: 92377 lines, 0 ignored, 2 matched, 92375 missed
[processed in 7.51 sec]
```
The final step is to activate the jail, add the following to `fail2ban/jail.local`:
```nginx
[nextcloud]
enabled = true
port    = http,https
filter  = nextcloud
logpath = /nextcloud/nextcloud.log
action  = iptables-allports[name=nextcloud]
```
The logpath is slightly different for applications that have multiple log files with dates:
```nginx
[jellyfin]
enabled  = true
filter   = jellyfin
port     = http,https
logpath  = /jellyfin/log*.log
action   =  iptables-allports[name=jellyfin]
```

Repeat the process for every external application, you can find Fail2Ban configurations for most applications on the internet.

If you need to unban an IP address that was blocked, run the following command on the docker host:
```
docker exec swag fail2ban-client unban <ip address>
```

This great mod sends a discord notification when Fail2Ban blocks an attack: [f2bdiscord](https://github.com/linuxserver/docker-mods/tree/swag-f2bdiscord).

## Geoblock
Geoblock reduces the attack surface of SWAG by restricting access based on countries.

Enable geoblock using either [DBIP mod](https://github.com/linuxserver/docker-mods/tree/swag-dbip) or [Maxmind mod](https://github.com/linuxserver/docker-mods/tree/swag-maxmind), follow the mod's instructions to set it up.

The mods come with 3 definitions for `$geo-whitelist`, `$geo-blacklist`, `$lan-ip`.

An example for allowing a single country:
```Nginx
map $geoip2_data_country_iso_code $geo-whitelist {
    default no;
    UK yes; #Replace with your country code list https://dev.maxmind.com/geoip/legacy/codes/iso3166/
}
```
An example for blocking high risk countries: (GilbN's list based on the Spamhaus statistics and Aakamai’s state of the internet report)
```Nginx
map $geoip2_data_country_iso_code $geo-blacklist {
    default yes; #If your country is listed below, remove it from the list
    CN no; #China
    RU no; #Russia
    HK no; #Hong Kong
    IN no; #India
    IR no; #Iran
    VN no; #Vietnam
    TR no; #Turkey
    EG no; #Egypt
    MX no; #Mexico
    JP no; #Japan
    KR no; #South Korea
    KP no; #North Korea
    PE no; #Peru
    BR no; #Brazil
    UA no; #Ukraine
    ID no; #Indonesia
    TH no; #Thailand
 }
```

Utilize the geoblock in your configuration by adding one of the following lines above your location section in every application you want to protect.

**Note that when using a whitelist filter, you also need to check if the source is a LAN IP, it's not required when using a blacklist filter.**
```nginx
    if ($lan-ip = yes) { set $geo-whitelist yes; }
    if ($geo-whitelist = no) { return 404; }
```
Or
```nginx
    if ($geo-blacklist = no) { return 404; }
```

Example:

```Nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name authelia.*;
    include /config/nginx/ssl.conf;
    client_max_body_size 0;

    if ($lan-ip = yes) { set $geo-whitelist yes; } #Check for a LAN IP
    if ($geo-whitelist = no) { return 404; } #Check the country filter

    location / {
        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        set $upstream_app authelia;
        set $upstream_port 9091;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;
    }
}
```

Add the lines to every external application based on your needs.


## NGINX Configuration
### X-Robots-Tag
You can prevent applications from appearing in results of search engines and web crawlers, regardless of whether other sites link to it. It doesn't work on all search engines and web crawlers, but it significantly reduces the amount.

Add the X-Robots-Tag config line to `ssl.conf` to enable it on **all** of your applications:
```
add_header X-Robots-Tag "noindex, nofollow, nosnippet, noarchive";
```

Disable on a specific application and allow search engines to display it by add the following line to the application config inside the server tag:
```
add_header X-Robots-Tag "";
```

### HSTS
HTTP Strict Transport Security (HSTS) is a web security policy mechanism that helps to protect websites against man-in-the-middle attacks such as protocol downgrade attacks and cookie hijacking. It allows web servers to declare that web browsers (or other complying user agents) should automatically interact with it using only HTTPS connections, which provide Transport Layer Security (TLS/SSL), unlike the insecure HTTP used alone.

**HSTS requires a working SSL certificate on your domains before enabling it.**

Enable HSTS by uncommenting the HSTS config line in ssl.conf:
```
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

#### Optional - Strengthening HSTS
After enabling the HSTS header, users are still vulnerable to attack if they access an HSTS‑protected website over HTTP when they have:

- Never before visited the site
- Recently reinstalled their operating system
- Recently reinstalled their browser
- Switched to a new browser
- Switched to a new device (for example, mobile phone)
- Deleted their browser’s cache
- Not visited the site recently and the max-age time has passed

To address this, Google maintains a “HSTS preload list” of web domains and subdomains that use HSTS and have submitted their names to [HSTS Preload](https://hstspreload.org/). This domain list is distributed and hardcoded into major web browsers. Clients that access web domains in this list automatically use HTTPS and refuse to access the site using HTTP.

Be aware that once you set the STS header or submit your domains to the HSTS preload list, it is impossible to remove it. It’s a one‑way decision to make your domains available over HTTPS.


## Authelia
Authelia is an open-source authentication and authorization server providing 2-factor authentication and single sign-on (SSO) for your applications via a web portal. Refer to this [blog post to configure Authelia](https://blog.linuxserver.io/2020/08/26/setting-up-authelia/).


