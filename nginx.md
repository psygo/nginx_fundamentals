# nginx

[NGINX Fundamentals: High Performance Servers from Scratch | Udemy](https://www.udemy.com/course/nginx-fundamentals/)

## About NGINX

- Developed by Igor Sysoev in 2005 out of frustrations with Apache.
- It can handle tens of thousands of concurrent connections.
- Has quickly surged in popularity.
- It's usually used as a web server, but it's actually a broader type of server: reverse proxy server.

## Nginx vs Apache

- In Apache, each request is unique and done separately. Nginx can deal with requests asynchronously.
- Nginx cannot handle embedded languages, it needs a separate process.
    - Nginx doesn't need server-side languages.
- Nginx interprets incoming requests as URI locations whereas APache prefers to interpret requests as filesystem locations.

## Installation

The preferred way of doing it is installing it from source.

nginx.com is the flashier website of nginx.org.

Use systemd to add nginx as a service.

## Understanding Configuration Terms

The 2 main configuration terms:

- `Context`
    - Scope
    - Inheritance
- `Directive`
    - Standard
        - Can only be declared once. A seconde declaration overrides the first.
        - Gets inherited by all child contexts.
        - Child context can override inheritance by re-declaring directive.
        - Example: `root /sites/site2`
    - Array
        - Can be declared many times
        - Example: `access_log`
        - Redeclarations in children override their parents
    - Action
        - Invokes an action sucha as a rewrite or a redirect
        - Inheritance doesn't apply as the request is either stopped (redirect/response) or re-evaluated (rewrite)
        - Example: `return 403 "You do not have permission to view this."`

### A Simple Example

```nginx
events {}

http {
  types {
    text/html html;
    text/css css;
  }

  server {
    listen 80;
    server_name 167.99.93.26;

    root /sites/demo;
  }

  # Prefix match
  # Equals match with `=`
  # Regex match (with the pcre library) with `~`
  # E.g.: `~* /greet[0-9]`
  # Regex has higher priority. But you can override this with `^`
  location /greet {
    return 200 'Hello from NGINX greet location'
  }

  # Variables can be strings, integers or booleans
  set $weekend 'No';

  # Conditional Example
  # Check static API key
  if ($arg_apikey != 1234) {
    return 401 "Incorrect API key"
  }

  if ($date_local ~ 'Saturday|Sunday') {
    set $weekend 'Yes';
  }

  # An example of variable usage
  location /inspect {
    return 200 '$host\n$uri\n$args'
    # Or 'Name: $arg_name';
  }

  # Redirection example
  location /logo {
    return 307 /thumb.png;
  }

  # Rewrites are done internally only (the URL is the same to the user)
  # Requires more resources
  rewrite ^/user/\w+ /greet/$1;

  # Tries files sequentially
  # Use `@` to name a path
  try_files /thumb.png /greet @friendly_404;

  location @friendly_404 {
    return 404 "Sorry, that file could not be found.";
  }

  location /secure {
    access_log /var/log/nginx/secure.access.log;
    access_log /var/log/nginx/access.log;
    # access_log off;
    return 200 "Welcome to the secure area.";
  }
}
```

You could also replace the manual insertion of types with `include mime.types`.

Priority:

1. Exact match
1. Preferential Prefix Match
1. Regex Match
1. Prefix Match

## Logging

Nginx provides 2 types of logs:

- Error logs
    - 404's usually don't get listed here actually.
        - Only ill-handled 404's will end up here.
- Access logs
    - `access_log`

## Dealing with Server-Side Processing

### Worker Processes
### Buffers & Timeouts

Buffer sizes are more dependent on the requests than the server itself. If you don't know what you're doing, leave them alone.

The `sendfile` and `tcp_nopush` configurations are likely to be the most worthwhile for static websites.

### Adding Dynamic Modules

Through `nginx -V`, you can extract the current configuration, which will be very useful for not changing anything.

```nginx
# This matches the permissions from the request with the user in nginx.
user www-data;

# pid /var/run/new_ngingx.pid

# This doesn't equate to more performance, because nginx is asynchronous anyway.
# The typical configuration is to match the number of workers to the number of CPUs (use `nproc` or `lscpu`).
# `auto` does this automatically.
worker_processes auto;

events {
  # We can't simply increase this at will. It's directly proportional to the number of CPUs.
  # To find out the max number, use `ulimit -n`.
  # Max Connections = Worker Processes x Worker Connections
  worker_connections 1024;
}

http {
  include mime.types;

  server {
    listen 80;
    server_name 167.99.93.26;

    root /sites/demo;

    index index.php index.html;

    location / {
      try_files $uri $uri/ =404;
    }

    location ~\.php$ {
      # Pass php requests to the php-fpm service (fastcgi)
      include fastcgi.conf;
      # Either a UNIX socket or TCP/IP
      fastcgi_pass unix:/run/php/php7.1-fpm.sock;
    }
  }
}
```
## Performance

```nginx
http {
  gzip on; # anyone can override this
  gzip_comp_level 3;
  gzip_types text/css;
  gzip_types text/js;

  location ~* \.(css|js|jpg|png)$ {
    access_log off;
    add_header Cache-Control public;
    add_header Pragma public;
    add_header Vary Accept-Encoding; # encoding is optional
    expires 1M; # 1 Month
  }
}
```

You can even use `gzip` to compress the exchanges.

FastCGI's protocol can be used to create many cache variations.

### Moving to HTTP2

> Requires SSL.

- HTTP1 is textual; while HTTP2 is binary
- Compresses headers
- Persistent connections
- Multiplex Streaming
- Server push

You're gonna need to use `--with -http_v2_module`.

> HTTP2 is only available on top of SSL.

And you're gonna need to generate a new SSL certificate.

Add:

```nginx
http {
  listen 443 ssl http2;

  server {
    listen 80;
    server_name 167.99.93.26;
    # Redirecting to HTTPS
    return 301 https://$host$request_uri;
  }

  ssl_certificate /etc/nginx/ssl/self.crt;
  ssl_certificate_key /etc/nginx/ssl/self.key;

  # Disable SSL in favor of TLS
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers <ciphers>;

  # Enable Diffie-Hellman
  ssl_dhparam /etc/nginx/ssl/dhparam.pem;

  # Enable HSTS (never accept HTTP)
  add_header Strict-Transport-Security "max-age=31536000" always;

  # SSL sessions
  ssl_session_cache shared:SSL:40m;
  ssl_session_timeout 4h;
  ssl_session_tickets on;

  # Guarantees everything will be served at the same time
  location = /index.html {
    http2_push /style.css;
    http2_push /thumb.png;
  }
}
```

## Security

- Security - Brute Force Protection
- Reliability - Prevent Traffic Spikes
- Shaping - Service Priority

Siege is another tool for benchmarking.

```nginx
# Define limit zone
# Good for login forms
limit_req_zone $request_uri zone=MYZONE:10m rate=60r/m;
```

[NGINX Rate Limiting](https://www.nginx.com/blog/rate-limiting-nginx/)
[NGINX rate-limiting in a nutshell](https://www.freecodecamp.org/news/nginx-rate-limiting-in-a-nutshell-128fe9e0126c/)

### Basic Auth (OAuth)

First install `yum install httpd-toold`.

Creating a user with a password:

```sh
htpasswd -c /etc/nginx/.htpasswd user1
```

Then adding basic auth to nginx:

```nginx
location / {
  auth_basic "Secure Area";
  auth_basic_user_file /etc/nginx/.htpasswd;
}
```

### Hardening NGINX

`server_tokens off;` might help.

```nginx
# Dealing with CORS in `<iframe>`s with user info
add_header X-Frame-Options "SAMEORIGIN";
add_header X-XSS-Protection "1; mode=block"; # cross-site scripting
```

> Is this the way to hack someone with HTML?

Installing NGINX via a package manager does not allow us to remove default modules.

### SSL Certificates

*Let's Encrypt* and *certbot* can both be used to generate SSL certificates.

## Reverse Proxy

A reverse proxy is an intermediary between the client and the service.

```nginx
events {}

http {
  server {
    listen 8888;

    location {
      return 200 "Hello from NGINX\n";
    }

    location /php {
      # The PHP process is on port 9999
      proxy_pass http://localhost:9999/
    }
  }
}
```

## Load Balancing

```nginx
events {}

http {
  # Defaults to round robin
  upstream phpservers {
    ip_hash;
    server localhost:10001;
    server localhost:10002;
    server localhost:10003;
  }

  server {
    listen 8888;

    location / {
      proxy_pass http://phpservers;
    }
  }
}
```

## Documentation

- The section on rewrite rules is specially important.
- The module reference is basically the real documentation.
- The common pitfalls is a must-read for any real service.
- Digital Ocean and WordPress has a lot of useful guides.
- This is a very helpful resource: [fcambus/nginx-resources: A collection of resources covering Nginx, Nginx + Lua, OpenResty and Tengine](https://github.com/fcambus/nginx-resources)

## Extras

- There's a GeoIP module for knowing from where the connection is coming.
- It also offers modules for streaming videos.
