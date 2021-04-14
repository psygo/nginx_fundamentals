# nginx

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
    - Scope.
- `Directive`

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
  try_files /thumb.png /greet;
}
```

You could also replace the manual insertion of types with `include mime.types`.

Priority:

1. Exact match
1. Preferential Prefix Match
1. Regex Match
1. Prefix Match
