

# Demon Deployment
You've built your application and are ready to deploy it live! This tutorial will give you recommendations on what you should do to setup your production environment to run a WebSocket server. None of these are required, but recommended. Please note this page serves as an introduction and boilerplate setup for each technoloogy; each topic has wealths of more detailed resources available.

## ulimit
Before you run your shell script (your WebSocket application) you should probably update your ulimit. What is this? It's a unix security feature that limits the number of open file descriptors one running process can have open at a time. We want to increase this as each client connecting to your server opens a file descriptor. You should update this number to make sure the number of clients you expect never exceeds your ulimit. Remember, when you update your ulimit, that change only affects the current shell session. Before you run you application make sure to execute this command:
```
$ ulimit -n 10000
```

## Libevent
Libevent is an asynchronous event driven C library that is meant to replace default event loop API on your system. It presents a common API that will utilize any event loop that your kernel uses. To set your system to use Libevent you will need to install the system library, the development tools, and the PHP extension:
```
$ sudo apt-get install libevent libevent-dev
$ sudo pecl install libevent
```
The event loop that PHP uses by default through stream_select uses the old, slow poll mechanism. By using Libevent PHP will use the faster epoll or kqueue, drastically improving concurrency (handling many connections quickly). Once Libevent is installed you don't need to change anything in your script. Ratchet's IoServer::factory will automatically use Libevent if it's available.

## XDebug
Disable the XDebug extension. Make sure it is commented out of your php.ini file. XDebug is fantastic for development, but destroys performance in production.

## Server Configuration
There's a couple things you need to know when deploying a WebSocket server that impact how you set up your server architecture:

* Apache can not handle or even pass-through WebSocket traffic (as of this writing)
* Some aggressive proxies will block traffic that isn't on port 80 or 443 (not many, research your target audience)

Given these issues with WebSockets we have three choices on how to architect:

* Run your website and WebSocket server on the same machine using port 8080 for WebSockets and take the chance client proxies won't block traffic
* Run your WebSocket server on its own server on port 80 under a subdomain (sock.example.com)
* Put a reverse proxy (HAProxy or Varnish) in front of your webserver and WebSocket server

The first two options are fairly easy with the second being a decent option if you can afford a second server. This article will detail the third option and show you how to configure your network, specifically HAProxy, to run your web and WebSocket servers on the same machine. We've chosen to demonstrate HAProxy because it can also handle SSL (in v1.5) where Varnish can not. The goal is to setup the architecutre like the diagram below.



As of this writing, HAProxy support for WebSockets (via tunnel mode) (and SSL) is in the unstable branch, so we'll have to install it from source:

```
wget http://haproxy.1wt.eu/download/1.5/src/devel/haproxy-1.5-dev17.tar.gz
tar -zxvf haproxy-1.5-dev17.tar.gz
cd haproxy-1.5-dev17
make install
```

The default installation should be in `/usr/local/sbin/haproxy`. The next step is to configure it. Below is a sample configuration:

```
global
	log	127.0.0.1	local0
	maxconn	10000
	user	haproxy
	group	haproxy
	daemon

defaults
	mode			http
	log			global
	option			httplog
	retries			3
	backlog			10000
	timeout	client		30s
	timeout	connect		30s
	timeout	server		30s
	timeout	tunnel		3600s
	timeout	http-keep-alive	1s
	timeout	http-request	15s

frontend public
	bind		*:80
	acl		is_websocket hdr(Upgrade) -i WebSocket
	use_backend	ws if is_websocket #is_websocket_server
	default_backend	www

backend ws
	server	ws1	127.0.0.1:1337

backend www
	timeout	server	30s
	server	www1	127.0.0.1:1338
```

Save this file, `/etc/haproxy.cfg` for example. You'll notice we set the user an group to haproxy; you'll need to create these or update your configuration to use a different user/group combo. Note the important parts above begin at "frontend public". We're telling HAProxy to listen on port 80 and if we find an HTTP header with the "Upgrade: WebSocket" direct it to our WebSocket application running on port 1337 and all other traffic to port 1338 running our traditional web stack (Nginx or Apache). Finally, run HAProxy:

```
$ sudo haproxy -f /etc/haproxy.cfg -p /var/run/haproxy.pid -D
```

## Supervisor
When running Ratchet in production it's highly recommended launching it from suporvisord. Suporvisor is a daemon that launches other processes and ensures they stay running. If for any reason your long running Ratchet application halted the supervisor daemon would ensure it starts back up immediately. Supervisor can be installed with any of the following tools: pip, easy_install, apt-get, yum. Once supervisor is installed you generate a template file with the following command:
```
$ echo_supervisord_conf > supervisor.conf
```
The following is a modification from the generated supervisor.conf file to run a Ratchet application from the "Hello World" tutorial:
```
[unix_http_server]
file = /tmp/supervisor.sock

[supervisord]
logfile          = ./logs/supervisord.log
logfile_maxbytes = 50MB
logfile_backups  = 10
loglevel         = info
pidfile          = /tmp/supervisord.pid
nodaemon         = false
minfds           = 1024
minprocs         = 200

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl = unix:///tmp/supervisor.sock

[program:ratchet]
command                 = bash -c "ulimit -n 10000 && /usr/bin/php ./bin/tutorial-terminal-chat.php"
process_name            = Ratchet
numprocs                = 1
autostart               = true
autorestart             = true
user                    = root
stdout_logfile          = ./logs/info.log
stdout_logfile_maxbytes = 1MB
stderr_logfile          = ./logs/error.log
stderr_logfile_maxbytes = 1MB
```
If you're only going to user supervisor to run your WebSocket application you can now run it with the command:
```
$ sudo supervisord -c supervisor.conf
```
## Links
[LibEvent](http://libevent.org/)
[Libevent PHP extension](http://pecl.php.net/package/libevent)
[HAProxy Configuration Manual v1.5](http://haproxy.1wt.eu/download/1.5/doc/configuration.txt)
[Supervisor Configuration](http://supervisord.org/configuration.html)
