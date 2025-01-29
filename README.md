# FLMP
FrankenPHP Linux MySQL PHP - a docker composer development environment

This is a dockerised FLMP stack solution including FrankenPHP, MySQL, PHP and Caddy.
It is additionally configured to allow connecting to and between a locally running service on port 8080 (which could e.g. be a VueJS front-end SPA).

## Overview

The architecture here is as follows. Docker creates a network between two containers: FrankenPHP and MySQL. FrankenPHP is a Caddy webserver module, which means it has Caddy webserver and PHP built in. The job of Caddy is to map network routes for the PHP back-end and Javascript front-end. FrankenPHP has it's own instance of PHP which is separate to that available on the CLI within the image. Also since FrankenPHP is managed by systemd, it has it's own /tmp folder which is not the system /tmp folder.

## Raison d'être

Localhost dev environments have become a bit more complicated due to recent changes in browser CORS policies, the result of which is that session cookies will not be stored by the browser on localhost unless the TCP protocol in use is https.

### Why FrankenPHP?

- FrankenPHP is a Caddy module, written in Go!, which is multi-threading and lightning fast.
- It handles https automagically, including generating all the necesary certificatees for localhost
- You can do everything Apache or Nginx can but the configuration is much simpler
- It's also significantly faster than php-fpm or cgi

### Caddyfile (config)

Look at the Caddyfile to see the webserver config. It's just a few lines of JSON. There are three main directives:

- debug: just provides more verbose output to stdout on the docker process
- fe.dev.localhost: defines the reverse proxy to whatever is listening on http port 8080 (e.g. your VueJS app running as a dev server)
- be.dev.localhost: defines the PHP back-end listener

Note that the two domains (front-end and back-end) must be fully qualified; i.e. they must contain at least two dots. You can make them whatever you want as long as they contain two dots (i.e. min. 3-parts to the domain name) and provided that the last part is `localhost`.

## Installation

In order to see this in action, you need the following:

1. Create an (e.g.) index.php file (or checkout a working PHP api) into the folder defined in the Caddyfile `root` directive for the back-end listener.
1. Launch the containers with the `docker-compose up` CLI command, run from the root of this project (i.e. the folder this README file is in).
1. Add the generated root cert file to your browser's trusted store cache. This can be found in `caddy/data/caddy/pki file
1. Run `curl` or browse to your back-end domain to see that in action
1. Start your front-end dev server listening on port 8080
1. Browse to your front-end FQDN (NB: not http://localhost:8080) to see the front-end in action


## Project Structure

```
.

├── db/
│   └── Dynamically generated MySQL database files
│
├── frankenphp/
│   ├── Caddyfile          # Caddy server configuration
│   └── Dockerfile         # FrankenPHP container setup
│
├── app/ (E.g.)
│   ├── be/            # Backend application
│   │   ├── config/        # Backend configuration
│   │   ├── plugins/       # Custom plugins (XeroPlugin, Cron, etc.)
│   │   ├── src/          # Main application source code
│   │   │   ├── Controller/   # API Controllers
│   │   │   ├── Model/       # Database models and entities
│   │   │   └── ...
│   │   └── ...
│   │
│   ├── fe/            # Frontend application
│   │   ├── src/           # Source code
│   │   ├── env/           # Environment configurations
│   │   └── ...
│   │
│   └── dist/       # Frontend distribution files
│
├── docker-compose-php8.yml  # Docker compose for PHP 8.x
```

### Key Components

- **FrankenPHP**: Current web server setup using Caddy with PHP integration
- **Database**: MySQL database (files generated in db/ directory)
