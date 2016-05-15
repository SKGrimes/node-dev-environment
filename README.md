# node-dev-environment
setting up a basic node developer environment on mac os x

You have to export the server components



```javascript
//routes.js
exports.register = (server, options, next) => {

    server.route({
        method: 'GET',
        path: '/test',
        handler: (request, reply) => {

            return reply('ok');
        },
    });

    return next();
};


exports.register.attributes = {
    name: 'test',
    version: '1.0.0',
};


```



Then require them



```javascript
const Routes = require('./routes.js');
// Create a server with a host and port
const server = new Hapi.Server();
server.connection({
    host: 'localhost',
    port: 9000
});

// Add the route
server.route({
    method: 'GET',
    path:'/',
    handler: function (request, reply) {

        return reply('hello world');
    }
});

server.register({
    register: Routes,

})

// Start the server
server.start((err) => {

    if (err) {
        throw err;
    }
    console.log('Server running at:', server.info.uri);
});

```

# API-Bridge
![David](https://david-dm.org/skgrimes/api-bridge.svg)
![Travis-CI](https://travis-ci.org/SKGrimes/api-bridge.svg?branch=master)
![A Bridge](http://imgur.com/ufVzhG6.png)

A HapiJS API gateway & clearing house to register internal services. The api-bridge is meant to be the first point of contact after the DNS routing resolves your IP to your hosting provider. The main business application logic for routing and resource dispersement it will handle the authentication, authorization and internal policies governing the subset of internal services the application manages.

## How it fits in, e.g. basic context
The [netflix micro-service guide](https://www.nginx.com/blog/microservices-at-netflix-architectural-best-practices/) lays some solid ground work for perfecting micro-service installations. However, not to sound super defeatest, but it is statically unlikely that more than one other org [will have 37.05% of America's peak downstream U.S. traffic]. This is meant to be for a handling reasonable amounts of traffic and services without over-complicating the process. Applications can receive an inbound request after it is initialized by some client, in the following manner. Let's assume some random link to `example.com/some route`. The DNS would resolve to a static IP address for the main application, I like digital ocean, and this is what they call a [floating ip](https://www.digitalocean.com/community/tutorials/how-to-use-floating-ips-on-digitalocean), but it is pretty standard. That static IP, allows load balancing and remapping without having to reconfigure DNS on the fly. This internal static IP would point to a private IP. If it is neccessary,  something like a [private network](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-use-digitalocean-private-networking) could receive the proxied request, but it doesn't matter. Either way a request is hitting the main server/infrastructure. The actual api-bridge would sit behind a server or reverse proxy. Using something like (NginX)(https://www.nginx.com) or [haproxy](ha proxy) monitoring the main inbound ports e.g. `example.com:80` and `example.com:443` and re-route requests to the internal bridge. This would handle  proper SSL termination inside the application. The application itself does have internal encrypton/ssl, see the note below on generating certs to secure the application.


## Setting Up The Development Env
To test out basic functionality and get at least a somewhat similar sense for how this could look in production there are some reasonably easy steps to mirror the deployment configuration. This assumes running OS X on a Mac and deploying into a linux environment. On linux, setup would be similar and even closer to what would be replicated in production. Configuration setup:

1. DNS Resolution with DNSMasq
2. Nginx reverse proxy configuration
3. SSL cert setup

###DNSMasq
[Dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) does a lot of things, from the docs, "provides network infrastructure for small networks: DNS, DHCP, router advertisement and network boot". You can use it to configure a subdomain and internal DNS resolution. Setting up DNS on the local environment just makes it a bit easier to work with the technology stack as it would exist in production. That is the reason it is employed here. This would typically just be done on the hosting provider or some DNS hosting service panel. To get started use homebrew and download dnsmasq.

```
brew update && brew install dnsmasq

cp $(brew list dnsmasq | grep /dnsmasq.conf.example$) /usr/local/etc/dnsmasq.conf

sudo cp $(brew list dnsmasq | grep /homebrew.mxcl.dnsmasq.plist$) /Library/LaunchDaemons/

sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist
```

This will install dnsmasq, copy the `dnsmasq.config` file to the local `/ect` directory and start the service now, and subsequently at system startup automatically, so it won't have to be done manually. On OS X el Capitan there seems to be permissions issues with the new SIP security.

**Note** Setting up the configs properly basically requires `sudo` everywhere on el capitan and making sure the files have correct permissions as some are symlinked or otherwise just not read and fail silently causing this not to work. Also, the default homebrew directory is assumed to be `/usr/local`.


Add the following to the file `/usr/local/etc/dnsmasq.conf`.

```

address=/.self/127.0.0.1
address=/.world/127.0.0.1
address=/.dev/127.0.0.1
```

Whatever extensions you add here e.g. the `/.self` extension above, need corresponding files in the `/ect/resolver` directory which you will need to make if it doesn't exist.

```
# make a resolver dir
sudo mkdir /ect/resolver

# make a file for all extensions you added above
sudo touch /ect/resolver/self /ect/resolver/dev /ect/resolver/world


sudo nano /etc/resolver/self
# add localhost ip to all files, e.g. from nano:
nameserver 127.0.0.1

# repeat for each domain
```

The above files are basically representative of a toplevel domain on the internet. Dnsmasq acts as a dns server to some extent (although not the same exact way) and resolves domains locally. After that is complete, confirm dnsmasq is running.

```
 ps ax | grep dnsmasq

 2160   ??  Ss     0:00.01 /usr/local/opt/dnsmasq/sbin/dnsmasq --keep-in-foreground -C /usr/local/etc/dnsmasq.conf
 2187 s001  S+     0:00.00 grep --color=auto dnsmasq

```

...and that the resolution is happening properly.

```
ping hello.self

PING hello.self (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.038 ms
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.059 ms
^C
--- hello.self ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.038/0.049/0.059/0.010 ms

klevvver@221b /etc
❯

```


Add the service alias to `~/.bash_profile` or `~/.alias`  if you'd like

then stop dnsmasq `sudo launchctl stop homebrew.mxcl.dnsmasq` and start it again `sudo launchctl start homebrew.mxcl.dnsmasq`

```
# start dnsmasq
alias strdnsmasq="sudo launchctl stop homebrew.mxcl.dnsmasq"

# stop
alias stpdnsmasq="sudo launchctl start homebrew.mxcl.dnsmasq"

# restart dnsmasq
alias rsdnsmasq="sudo launchctl stop homebrew.mxcl.dnsmasq && sudo launchctl start homebrew.mxcl.dnsmasq"
```


## Nginx

![nginx](https://assets.wp.nginx.com/wp-content/uploads/2015/04/NGINX_logo_rgb-01.png)


With homebrew install nginx and follow the directions after prompt. On completion of setting up launchd open with nano or editor the file, `/usr/local/etc/nginx/nginx.conf` which is nginx config file.

**note** The http2 module isn't nec. but a realistic addition, you can uninstall and reinstall nginx to be built with HTTP2 support by throwing the flag `--with-http2`. Additionally `--with-libressl` but openssl is fine.

```
❯ brew install  nginx --with-http2
==> Downloading http://nginx.org/download/nginx-1.10.0.tar.gz
######################################################################## 100.0%
==> ./configure --prefix=/usr/local/Cellar/nginx/1.10.0 --with-http_ssl_module --with-pcre --with-ipv6 --sbin-pa
==> make install
==> Caveats
Docroot is: /usr/local/var/www

The default port has been set in /usr/local/etc/nginx/nginx.conf to 8080 so that
nginx can run without sudo.

nginx will load all files in /usr/local/etc/nginx/servers/.

To have launchd start nginx now and restart at login:
  brew services start nginx
Or, if you don't want/need a background service you can just run:
  nginx
```


```
sudo cp /usr/local/opt/nginx/*.plist /Library/LaunchDaemons
sudo launchctl load -w /Library/LaunchDaemons/homebrew.mxcl.nginx.plist

nano /usr/local/etc/nginx/nginx.conf
```

If that file doesn't exist there should be a default.conf file in that toplevel dir. After that make sure nginx is running.

```
pa ax grep | grep nginx
sudo nginx
```

**Main Static Root Directory**

If installed with homebrew the file should be inside the cellar inside nginx directory. version numbers differ and there is no SSL support by default so http:localhost:3000 will succeed but `https://` will *not*.


```
usr/local/Cellar/nginx/1.10.0/html
```

The above directory path is what nginx will resolve to by default. However, the setup  uses proxy pass for local development.

![Nginx default](https://dab1nmslvvntp.cloudfront.net/wp-content/uploads/2013/09/nginx-fcgi-2.png)

### Config file /usr/local/ect/nginx/nginx.conf
```

worker_processes  1;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen 80;

        server_name localhost;

        location / {
            proxy_pass 127.0.0.1:3333;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }


    include servers/*;
}

```

After restarting nginx,

```
$ sudo nginx -s stop && sudo nginx
```

If proxypass wasn't enabled yet, you can see the default message. Or it is possible you already saw it in a previous step. However, with proxypass we will generate a hello message from nginx which should show up on port 80, e.g. `http:localhost/` or `127.0.0.1`, or even the above domains setup with dnsmasq like `http://love.your.self`. ect.



###Basic Commands

```
# stop
sudo nginx -s stop

# start
sudo nginx

#restart
sudo nginx -s reload
```





### confirm it works with node

If you copy this and run it, then go to somedomain.self (or whatever extension) it should resolve to helloworld.


**server.js**
```
var port = 3333;
var host = 'localhost';

var http = require('http');
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(port, host);
console.log('Server running at http://',host, port);
```

Run `node server.js` and restart nginx `sudo nginx -s stop && sudo nginx`, if done properly these should work:

* `http://anything.in.resolvers.dev` should show up as if local host.
* nginx should pass any of the resolver domains from dnsmasq to `port:3333`, which shows `hello world`.
* http://something.self, http:hello.dev, http://127.0.0.1, localhost, ect should all resolve to the simple node server (or whatever is runnning on the port configured in the nginx.conf)

## Openssl

![Open SSl](http://openssl.com/images/openssl-logo.png)



We will make fake credentials to simulate a cert. They aren't fake, but they aren't accepted anywhere except locally, and maybe not even there depending on security...

```
$ sudo openssl genrsa -out "/usr/local/etc/ssl/absvrd.key" 2048

```

This will generate a fake key that will be inside of the`/usr/local/ect/ssl` directory. The prompt asks for basic info to generate a key.

**Note:** Annoyingly, chrome doesn't allow super wildcard certs like `*.domain`, so at the common name prompt a subdomain must be configured.

```

sudo openssl req -new -key "/usr/local/etc/ssl/absvrd.key" -out "/usr/local/etc/ssl/absvrd.csr"

You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:FU
State or Province Name (full name) [Some-State]:Some State
Locality Name (eg, city) []:so local, much host
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Skunk Works
Organizational Unit Name (eg, section) []:221b
Common Name (e.g. server FQDN or YOUR name) []:*.lokle.dev
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

Sign the cert.

```

sudo openssl x509 -req -days 60 \
    -in "/usr/local/etc/ssl/absvrd.csr" \
    -signkey "/usr/local/etc/ssl/absvrd.key" \
    -out "/usr/local/etc/ssl/absvrd.crt"

Signature ok
subject=/C=US/ST=RI/L=Little Compton/O=Next Massive Unicorn Startup Co./OU=skunkworks/CN=*.lokle.dev
Getting Private key

```

### Full Nginx Config file
```
/usr/local/etc/nginx/nginx.conf
```
contents:
```
###################################
#        NGINX CONFIG FILE        #
###################################

# Process Info
###############

#user  nobody;
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
#pid        logs/nginx.pid;



worker_processes  1;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen 80;

        server_name localhost;

        location / {
            proxy_pass http://localhost:3333;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }

    include servers/*;
}



```

### Nginx /servers/unicorn.dev config file

Inside of `/servers` referenced on last line, which is relative to the path of the `nginx.conf` make a file called `dangerous` or in reality whatever the host name is supposed to be, if in production, and use it to identify the additional ssl config.

The above proxy from to `http://127.0.0.1:3000` to localhost on 3333, e.g. `http://127.0.0.1:3333`




see basic config file below in appendix.


Inside file example. **unicorn.dev**

```
 server {

        listen   443;
        server_name unicorn.dev;


        location / {
            proxy_pass http://localhost:3333;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }

        ssl on;
        ssl_certificate /usr/local/etc/ssl/dangerzone.chained.crt;
        ssl_certificate_key /usr/local/etc/ssl/dangerzone.key;
}
```

copy cert to desktop & clear

```
cp /usr/local/etc/ssl/dangerous.crt ~/Desktop/
```
Double click cert on the desktop to launch keychain. The cert should be visible, and the top item which needs to have the trust configured and saved before it will work. You will need to save the settings after. If you exit, it will confirm update by askling
for your admin password and propogate settings.

These settings allow the browser to trust the cert, but not every bloody process.

![keychain setup](https://i.imgur.com/YSITAL1.png)
*note that the common name is incorrect in the picture, it should show something like `*.lokle.dev`.

Dump the dnscache.

```
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder; say DNS cache flushed
```

This will work in chrome:

![chrome](https://i.imgur.com/jrgx8vW.png)

##KeyMetrics PM2

Use pm2 to manage app. El Capitan permissions/SIP is super annoying. Kernal panic/cpu spikes shown in activity monitor were >80% cpu on startup. Pm2 should probbaly be run in dev mode.


**Developer Mode**

This monitors filechanges and is likely a better solution anyway. Nginx will likely need to be started manually.

```
npm install pm2 -g

sudo nginx
sudo pm2-dev start /path/to/app.js
```
The console should log status output and reboot on file changes.


**Possibly Will Not Work**
This should be how to automate pm2 on startup.

```
sudo npm install pm2 -g
sudo pm2 start /full/unix/path/to/app
sudo pm2 startup darwin
sudo launchctl load /Library/launchAgents/keymetricsScript/path

sudo pm2 status

```
Then if the app is still up, shutdown and reboot.
## Application Quickstart

$ node servers/api/server.js && node servers/gui/server.js
basics of that...

```
$ git clone [this repo]

$ npm install && npm start


$ sudo openssl genrsa -out "/Users/klevvver/Desktop/kamus/api-bridge/config/certs/key.key" 2048

sudo openssl req -new -key "/Users/klevvver/Desktop/kamus/api-bridge/config/certs/key.key" -out "/Users/klevvver/Desktop/kamus/api-bridge/config/certs/key.csr"

sudo openssl x509 -req -days 60 \
    -in "/Users/klevvver/Desktop/kamus/api-bridge/config/certs/key.csr" \
    -signkey "/Users/klevvver/Desktop/kamus/api-bridge/config/certs/key.key" \
    -out "/Users/klevvver/Desktop/kamus/api-bridge/config/certs/cert.crt"
```


## Stack

### HapiJS
 The server is built on [hapiJS](http://hapijs.com) which is a top-level application framework for the nodejs ecosystem that came out of Walmart's innovation laboratory. It was built by [eran hammer](https://hueniverse.com) who was a core contributor on oAuth and has some pretty colorful thoughts on both that project and security. Hapi is pretty secuity centric, and provides a much more workable standard than express which is [strongloop's](https://strongloop.com) web-application framework, that really isn't a framework. The Stack Overflow [answer to the what a librar v. framework is](http://stackoverflow.com/questions/148747/what-is-the-difference-between-a-framework-and-a-library)  serves as a fairly solid reference point. I'll assume you didn't read that and just give you my `$0.02`,  basically the difference is a framework is for configuration, and a library is for implementation. So for an application/server build there is just not a ton of need to write and control boilerplate code that is already built. A la cart plugins for basic feature sets:

 * [Boom](https://github.com/hapijs/boom) Error utility lib.


The snippet `Boom.unauthorized('invalid password', 'sample');` generates:

```
"payload": {
    "statusCode": 401,
    "error": "Unauthorized",
    "message": "invalid password",
    "attributes": {
        "error": "invalid password"
    }
},
"headers" {
  "WWW-Authenticate": "sample error=\"invalid password\""
}
```

 * [Hoek](https://github.com/hapijs/hoek) HapiJS Swiss Army knife, with a a bunch of useful utilities for the
 HapiJS ecosystem.

 * [Joi](https://github.com/hapijs/joi) Create immutable schema objects and use them for internal validation.

 * [Hapi-auth-basic](https://github.com/hapijs/hapi-auth-basic) && [hapi-auth-cookie](https://github.com/hapijs/hapi-auth-cookie) More in the Authentication section.


## Authentication

### SSL Certificate Generation
To securethe server configure it to use [Let's Encrypt](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04) to handle SSL certificates. It's actually *fairly* painless, and now a lot easier than it used to be. So assume, the above NginX router is using this for SSL/HTTPs. Then it proxy passes that to an internal port where the internal application sits.


## Issues

Flag em'

## To Do

So many things.

## Acknowledgements

 [HapiJS(obviously)](https://github.com/hapijs/hapi)

 [HapiJS University](https://github.com/hapijs/university)

 [Pete Coop's Generator Express](https://github.com/petecoop/generator-express)

 [AirBnB's JS Styleguide](https://github.com/airbnb/javascript)

 [Servers for Hackers](https://serversforhackers.com/video/self-signed-ssl-certificates-for-development)

[This irrelevant(here) email from Linus](https://lkml.org/lkml/2012/12/23/75)
 ```
 moved to gitlabs.
 ```
