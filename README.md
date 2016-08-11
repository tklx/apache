# tklx/apache - web server
[![CircleCI](https://circleci.com/gh/tklx/apache.svg?style=shield)](https://circleci.com/gh/tklx/apache)

The [Apache][apache] HTTP Server Project is an effort to develop and maintain an open-source HTTP server for modern operating systems including UNIX and Windows. The goal of this project is to provide a secure, efficient and extensible server that provides HTTP services in sync with the current HTTP standards.

The Apache HTTP Server ("httpd") was launched in 1995 and it has been the most popular web server on the Internet since April 1996. It has celebrated its 20th birthday as a project in February 2015.

## Features

- Based on the super slim [tklx/base][base] (Debian GNU/Linux).
- Apache2 installed directly from Debian.
- Uses [tini][tini] for zombie reaping and signal forwarding.
- Includes ``EXPOSE 80 443``, so standard container linking will make it
  automatically available to the linked containers.
- Can be coupled with another container to provide SSL access and/or
  proxying.

## Usage

### Simple static site hosting

#### From host

```console
$ docker run --name some-apache -v /some/content:/var/www/html:ro -d tklx/apache
```

```console
$ docker run --name some-apache -v /some/content:/var/www/html:ro -v /some/config/file:/etc/apache/sites-available/000-default.conf:ro -d tklx/apache
```

#### From host (cleaner solution with Dockerfile)

```console
$ ls
html/ 000-default.conf Dockerfile
$ cat Dockerfile
FROM tklx/apache

COPY html /var/www/html
COPY 000-default.conf /etc/apache/sites-available/000-default.conf
$ docker build -t some-content .
$ docker run --name some-apache -d some-content
```

#### From another container

```console
$ docker run --name some-content -v /var/www/html some-content
$ docker run --name some-apache --volumes-from=some-content -d tklx/apache
```

### Exposing the port

#### Specific port

```console
$ docker run --name some-apache -d -p 8080:80 tklx/apache
```

#### Docker-chosen port
```console
$ docker run --name some-apache -dP tklx/apache
$ docker port some-apache
443/tcp -> 0.0.0.0:32770
80/tcp -> 0.0.0.0:32771
```

### Setting up HTTPS websites

```console
$ docker run --name some-certs -v /etc/ssl/private:ro -d cert-provider
$ docker run --name some-config -v /etc/apache/ -d config-provider
$ docker exec some-config cat /etc/apache/sites-enabled/www.example.com.conf
NameVirtualHost *:443

<VirtualHost *:443>
    ServerName www.example.com
    SSLEngine On

    SSLCertificateFile /etc/ssl/private/www.example.com;
    SSLCertificateKeyFile /etc/ssl/private/www.example.com.key;

    DocumentRoot "/var/www"
</VirtualHost>
$ docker run --name some-apache --volumes-from=some-certs --volumes-from=some-config -d tklx/apache
```

We recommend using the official [guidelines][apache-ssl] to set up your SSL server correctly.

### Setting up a reverse proxy

```console
$ docker run --name some-app -v /var/www -v /etc/apache/sites-available -d backend-app
$ docker run --name some-apache --volumes-from=some-app --link some-app:some-app -d tklx/apache
$ docker exec some-apache ls /etc/apache/sites-enabled/
some-app-site.conf
$ docker exec some-apache cat /etc/apache/sites-enabled/some-app-site.conf
<VirtualHost *:80>
    ServerName www.example.com

    DocumentRoot "/var/www"

    <FilesMatch "\.php$">
        SetHandler "proxy:fcgi://some-app/"
    </FilesMatch>
</VirtualHost>
```

### Setting up a reverse proxy with SSL termination

```console
$ docker run --name some-certs -v /etc/ssl/private:ro -d cert-provider
$ docker run --name some-app -v /var/www -v /etc/apache/sites-available -d backend-app
$ docker run --name some-apache --volumes-from=some-app --volumes-from=some-certs --link some-app:some-app -d tklx/apache
$ docker exec some-apache ls /etc/apache/sites-enabled/
some-app-site.conf
$ docker exec some-apache cat /etc/apache/sites-enabled/some-app-site.conf
<VirtualHost _default_:80>
    ServerName www.example.com
    DocumentRoot "/var/www"

    ProxyPreserveHost On

    <FilesMatch "\.php$">
        SetHandler "proxy:fcgi://some-app/"
    </FilesMatch>
</VirtualHost>
<VirtualHost _default_:443>
    ServerName www.example.com
    DocumentRoot "/var/www"

    ProxyPreserveHost On
    SSLEngine On

    SSLCertificateFile /etc/ssl/private/www.example.com;
    SSLCertificateKeyFile /etc/ssl/private/www.example.com.key;

    <Location />
        SSLRequireSSL
    </Location>

    <FilesMatch "\.php$">
        SetHandler "proxy:fcgi://some-app/"
    </FilesMatch>
</VirtualHost>
```

## Status

Currently on major version zero (0.y.z). Per [Semantic Versioning][semver],
major version zero is for initial development, and should not be considered
stable. Anything may change at any time.

## Issue Tracker

TKLX uses a central [issue tracker][tracker] on GitHub for reporting and
tracking of bugs, issues and feature requests.

[apache]: https://httpd.apache.org/
[apache-ssl]: https://httpd.apache.org/docs/2.4/ssl/ssl_howto.html
[base]: https://github.com/tklx/base
[tini]: https://github.com/krallin/tini
[gosu]: https://github.com/tianon/gosu
[semver]: http://semver.org/
[tracker]: https://github.com/tklx/tracker/issues
