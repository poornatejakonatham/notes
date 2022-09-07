# Deploy a Golang Web Application Behind Nginx

We started last week strong with a [foray into Golang](https://hackersandslackers.com/create-your-first-golang-app/), where we created a simple web app serving a "Hello world" route. For those of you who were enticed by this deviation from our regular programming, the next logical question you might have could be how to make this knowledge "useful" by making it accessible by other human beings.

We're going to build on our Golang momentum from last week to do just that: deploy a web application written in Go to a Linux host such as Ubuntu. We're going to cover everything from installing Go, creating a systemctl service, and configuring Nginx. All you need is a VPS.

## Installation

### Install Nginx

In case you haven't done so already, make sure your VPS has Nginx installed.

```sh
$ apt update
$ apt upgrade -y
$ apt install nginx
```

### Installing Go on Linux

We're going to install Go via source. Pick the version of Go that suits your Linux distro's needs from the [Go downloads page](https://go.dev/dl/). We'll download this to our `/tmp` folder, build the source, and move the built source to where it belongs:

```sh
$ cd /tmp
$ wget https://go.dev/dl/go1.19.linux-amd64.tar.gz
$ tar -xvf go1.19.linux-amd64.tar.gz
$ sudo mv go /usr/local
```

We've just unpacked the Go language and moved it to where Linux typically likes to keep its programming languages. This path is what Go refers to as the **GOROOT**, and its contents should look like this:

```sh
$ tree /usr/local/go -L 1
/usr/local/go
├── api
├── bin
├── codereview.cfg
├── CONTRIBUTING.md
├── doc
├── lib
├── LICENSE
├── misc
├── PATENTS
├── pkg
├── README.md
├── SECURITY.md
├── src
├── test
└── VERSION

8 directories, 7 files
```

## Add GOPATH and GOROOT to your Shell Script

We've installed and unpacked Go, but we haven't given our OS a way to recognize that we've done so just yet. We can accomplish this by modifying our shell script, which is typically called `.profile`:

```sh
$ vim ~/.profile
```

Here we'll add our **GOROOT** and **GOPATH** file paths. If you'll recall, **GOROOT** is where our OS looks for the Go programming language, and **GOPATH** is the working directory where we keep all Go projects and dependencies. I've chosen to set my **GOPATH** to `/go`, which is a directory we have yet to create:

```sh
export GOPATH=/go
export GOROOT=/usr/local/go
export PATH=$PATH:$GOPATH
export PATH=$PATH:$GOROOT/bin
```

Save your changes and activate the shell script:

```sh
$ source ~/.profile
```

We can now verify that everything's been installed:

```sh
$ go version
go version go1.19 linux/amd64
```

## Setting up our GOPATH & Project

We have to create our **GOPATH** manually, which is as simple as creating the following directories:

```sh
$ mkdir ~/go
$ mkdir ~/go/bin
$ mkdir ~/go/pkg
$ mkdir ~/go/src
```

Now we have a place to keep our Go projects! I'm going to pull down the "Hello world" project we created last week for convenience sake:

```sh
$ cd /go
$ go get github.com/hackersandslackers/golang-helloworld
```

I've chosen to clone the Github repo into my `/src` path as well, which leaves the structure of my **GOPATH** looking like this:

```sh
$ tree .
/go
├── /bin
│   └── golang-helloworld
├── /pkg
└── /src
    └── /github.com
        └── /hackersandslackers
            └── /golang-helloworld
                ├── README.md
                ├── go.mod
                ├── go.sum
                ├── golang-helloworld
                ├── main.go
                └── main_test.go
```

## Create an Nginx Config

You may have done this a few times before, but whatever. We're going to set up an Ngnix reverse proxy to listen on the port our app will be running on, which happens to be port **9100** in our case. Of course, we need to make sure this port is enabled first:

```sh
$ sudo ufw allow 9100
```

Create a configuration file in the Nginx `/sites-available` folder:

```sh
$ sudo vim /etc/nginx/sites-available/golang-helloworld.conf
```

We're going to drop the standard boilerplate for a reverse proxy here. The domain I happen to be using for this app is golanghelloworld.hackersandslackers.app. Replace this with the domain of your choice:

```
server {
   listen 80;
   listen [::]:80;

   server_name    golanghelloworld.hackersandslackers.app;

   location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $http_host;
        proxy_pass http://127.0.0.1:9100;
    }

    location ~ /.well-known {
        allow all;
    }
}
```

We activate this configuration by creating a sym link from our file in `sites-available` to `sites-enabled`:

```sh
$ sudo ln -s /etc/nginx/sites-available/golang-helloworld.conf /etc/nginx/sites-enabled/golang-helloworld.conf
```

Finally, these changes are applied upon Nginx restart. If the below produces no output, you're in the clear:

```sh
$ sudo service nginx restart
```

## SSL With Certbot

Adding SSL is arguably out of scope for what the point of this tutorial is, but whatever. [Certbot](https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx) makes adding SSL easy enough that we can blow through it in less than a minute.

Before we can actually install Certbot, we need to add the proper repositories:

```sh
$ sudo apt-get update
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository universe
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
```

Now we can install Certbot for real:

```sh
$ sudo apt-get install certbot python3-certbot-nginx
```

The Certbot CLI is able to accept an `--nginx` flag, which scans the configurations we've set up on our machine:

```sh
$ certbot --nginx
```

This will list every Nginx configuration you have on your machine and prompt you for which app you'd like to set up with SSL. My Ubuntu machine happens to host a bunch of sites. Feel free to check out any of them if you please:

```sh
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator nginx, Installer nginx

Which names would you like to activate HTTPS for?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: broiestbro.com
2: www.broiestbro.com
3: consider.pizza
4: stockholm.ghostthemes.io
5: hackersandslackers.app
6: hackersandslackers.tools
7: django.hackersandslackers.app
8: flaskblueprints.hackersandslackers.app
9: flasklogin.hackersandslackers.app
10: flasksession.hackersandslackers.app
11: flasksqlalchemy.hackersandslackers.app
12: flaskwtf.hackersandslackers.app
13: plotlydashflask.hackersandslackers.app
14: www.hackersandslackers.tools
15: hustlers.club
16: www.hustlers.club
17: toddbirchard.app
18: golanghelloworld.hackersandslackers.app
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel):
```

The configuration I'm looking for is #18. Selecting this will then prompt whether or not we'd like to redirect HTTP traffic to HTTPS, which is something you should definitely do (is there even a reason not to do this? Let me know in the COMMENTS BELOW and remember to smash that LIKE button).

```sh
Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```

Select option **2**.

Check out how Certbot has modified our original `golang-helloworld.conf` Nginx config:

```
server {

   server_name    golanghelloworld.hackersandslackers.app;

   location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $http_host;
        proxy_pass http://127.0.0.1:9100;
    }

    location ~ /.well-known {
        allow all;
    }

    listen [::]:443 ssl;
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/golanghelloworld.hackersandslackers.app/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/golanghelloworld.hackersandslackers.app/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

}

server {

    if ($host = golanghelloworld.hackersandslackers.app) {
        return 301 https://$host$request_uri;
    }

   listen 80;
   listen [::]:80;

   server_name    golanghelloworld.hackersandslackers.app;
    return 404;

}
```

That's what we like to see.

## Create a Systemctl Service

If you've never messed around with Systemctl services before, a "service" is something we want to run continuously on our server (for example, Nginx is a service in itself). We're going to create a service for our Go app to make sure our app is always running, even if our server is restarted:

```sh
$ vim /etc/systemd/system/golanghelloworld.service
```

The syntax for Linux services follows the `.ini` file format. There's a lot here which we'll dissect in a moment:

```
[Unit]
Description=golanghelloworld.hackersandslackers.app
ConditionPathExists=/go/src/github.com/hackersandslackers/golang-helloworld
After=network.target

[Service]
Type=simple
User=root
Group=root

WorkingDirectory=/go/src/github.com/hackersandslackers/golang-helloworld
ExecStart=/go/src/github.com/hackersandslackers/golang-helloworld/golang-helloworld

Restart=on-failure
RestartSec=10

ExecStartPre=/bin/mkdir -p /var/log/golang-helloworld
ExecStartPre=/bin/chown syslog:adm /var/log/golang-helloworld
ExecStartPre=/bin/chmod 775 /go/src/github.com/hackersandslackers/golang-helloworld/golang-helloworld

[Install]
WantedBy=multi-user.target
```

Here are the notable values being set above:

- `User` & `Group`: Probably not the best idea in the word, but this tells our service to run our app as the Ubuntu root user. Feel free to change this to a different Linux user.
- `WorkingDirectory`: The working directory that we'll be serving our app from.
- `ExecStart`: The compiled binary file of our Go project.
- `Restart` & `RestartSec`: These values are exceptionally useful for ensuring that our app is always up and available, even after crashing from unforeseen circumstances. These two values are telling our service to check if our app is running every 10 seconds. If the app happens to be down, our service will restart the app, hence the `on-failure` value for `Restart`.
- `ExecStartPre`: Each of these lines contains a command to run prior to starting our app. We set three such commands: the first two ensure that our app logs correctly to a directory called `/var/log/golang-helloworld`. The third (and more important) command sets permissions on our Go binary file.

Save your service file. Below we register the changes we've made to our system's services, start our new service, and `enable` our service to be run upon system startup:

```sh
$ systemctl daemon-reload
$ service golanghelloworld start
$ service golanghelloworld enable
```

Let's check to see how things went:

```sh
$ service golanghelloworld status
```

If you happen to be very lucky, you'll see a SUCCESS output like the following:

```sh
golanghelloworld.service - golanghelloworld.hackersandslackers.app
   Loaded: loaded (/etc/systemd/system/golanghelloworld.service; disabled; vendor preset: enabled)
   Active: active (running) since Fri 2020-05-29 01:57:02 UTC; 4s ago
  Process: 21454 ExecStartPre=/bin/chmod 775 /go/src/github.com/hackersandslackers/golang-helloworld/golang-helloworld (code=exited, status=0/SUCCESS)
  Process: 21449 ExecStartPre=/bin/chmod 775 /var/log/golang-helloworld (code=exited, status=0/SUCCESS)
  Process: 21445 ExecStartPre=/bin/chown syslog:adm /var/log/golang-helloworld (code=exited, status=0/SUCCESS)
  Process: 21444 ExecStartPre=/bin/mkdir -p /var/log/golang-helloworld (code=exited, status=0/SUCCESS)
 Main PID: 21455 (golang-hellowor)
    Tasks: 6 (limit: 4915)
   CGroup: /system.slice/golanghelloworld.service
           └─21455 /go/src/github.com/hackersandslackers/golang-helloworld/golang-helloworld
```

## Debugging Systemctl Services

The unfortunate truth about creating Linux services is that there are a lot of moving parts at play; I don't think I've ever gotten a new service to work on the first attempt without some sort of error. These errors range from permission errors, incorrect file paths, or port clashing. The good news is that each of these problems are individually simple to fix.

Your first line of defense to debugging services is `journalctl` to check the log output of any service:

```sh
$ journalctl -u golanghelloworld.service
```

An indispensable tool for debugging issues is the ability to `grep` for processes to see if they're running properly. We can search active processes by name or by port. If `journalctl` outputs an error that port 9100 is in use, you can find that process via the below:

```sh
$ ps aux | grep 9100
```

We can kill whichever process comes back with `kill -9 [PID]`.

Alternatively, we can check if our app is running by grepping by process name:

```sh
$ ps aux | grep golang-helloworld
```

If needed, we could kill the process with `pkill -9 golang-helloworld`.

If you need to make changes to your `.service` file, remember to run `systemctl daemon-reload` to pick up the changes, and `service golanghelloworld restart` to give it another go.

## Get Out There

I have faith you'll work out the kinks and successfully get your Go app up and running. It took me a bit of time myself, but my shitty hello world is up and live in all its glory [here](http://golanghelloworld.hackersandslackers.app/).

If you run into issues that you can't seem to solve, feel free to reach out. We'll work it out together.
