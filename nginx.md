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
