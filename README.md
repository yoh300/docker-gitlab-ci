# Table of Contents
- [Introduction](#introduction)
    - [Version](#version)
    - [Changelog](Changelog.md)
- [Supported Web Browsers](#supported-web-browsers)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
    - [Database](#database)
        - [MySQL](#mysql)
            - [Internal MySQL Server](#internal-mysql-server)
            - [External MySQL Server](#external-mysql-server)
        - [PostgreSQL]($postgresql)
            - [External PostgreSQL Server](#external-postgresql-server)
    - [Mail](#mail)
    - [Putting it all together](#putting-it-all-together)
    - [Available Configuration Parameters](#available-configuration-parameters)
- [Maintenance](#maintenance)
    - [SSH Login](#ssh-login)
- [Upgrading](#upgrading)
- [References](#references)

# Introduction
Dockerfile to build a GitLab CI container image.

## Version
Current Version: 4.3.0

# Supported Web Browsers

- Chrome (Latest stable version)
- Firefox (Latest released version)
- Safari 7+ (Know problem: required fields in html5 do not work)
- Opera (Latest released version)
- IE 10+

# Installation

Pull the latest version of the image from the docker index. This is the recommended method of installation as it is easier to update image in the future. These builds are performed by the **Docker Trusted Build** service.

```bash
docker pull sameersbn/gitlab-ci
```

Starting from GitLab CI version 4.3.0, You can pull a particular version of GitLab CI by specifying the version number. For example,

```bash
docker pull sameersbn/gitlab-ci:4.3.0
```

Alternately you can build the image yourself.

```bash
git clone https://github.com/sameersbn/docker-gitlab-ci.git
cd docker-gitlab-ci
docker build -t="$USER/gitlab-ci" .
```

# Quick Start
Before you can start the GitLab CI image you need to make sure you have a [GitLab](https://www.gitlab.com/) server running. Checkout the [docker-gitlab](https://github.com/sameersbn/docker-gitlab) project for getting a GitLab server up and running.

You need to provide the URL of the GitLab server while running GitLab CI using the GITLAB_URL environment configuration. For example if the location of the GitLab server is 172.17.0.2

```bash
docker run -name gitlab-ci -d \
  -e "GITLAB_URL=http://172.17.0.2" \
  sameersbn/gitlab-ci
GITLAB_CI_IP=$(docker inspect gitlab-ci | grep IPAddres | awk -F'"' '{print $4}')
```

Alternately, if the GitLab and GitLab CI servers are running on the same host, you can take advantage of docker links. Lets consider that the GitLab server is running on the same host and has the name **"gitlab"**, then using docker links:

```bash
docker run -name gitlab-ci -d -link gitlab:gitlab sameersbn/gitlab-ci
GITLAB_CI_IP=$(docker inspect gitlab-ci | grep IPAddres | awk -F'"' '{print $4}')
```

Access the GitLab CI server

```bash
xdg-open "http://${GITLAB_CI_IP}"
```

Login using your GitLab credentials.

You should now have GitLab CI ready for testing. If you want to use GitLab CI for more than just testing then please read the **Advanced Options** section.

**PS:** You need to install [GitLab CI Runner](https://gitlab.com/gitlab-org/gitlab-ci-runner/blob/master/README.md) if you want to do anything worth while with the GitLab CI server. Please look up github / docker index service for runner containers.

# Configuration

## Database
GitLab CI uses a database backend to store its data.

### MySQL

#### Internal MySQL Server
This docker image is configured to use a MySQL database backend. The database connection can be configured using environment variables. If not specified, the image will start a mysql server internally and use it. However in this case, the data stored in the mysql database will be lost if the container is stopped/removed. To avoid this you should mount a volume at /var/lib/mysql.

```bash
mkdir /opt/gitlab-ci/mysql
docker run -name gitlab-ci -d \
  -e "GITLAB_URL=http://172.17.0.2" \
  -v /opt/gitlab-ci/mysql:/var/lib/mysql sameersbn/gitlab-ci
```

This will make sure that the data stored in the database is not lost when the image is stopped and started again.

#### External MySQL Server
The image can be configured to use an external MySQL database instead of starting a MySQL server internally. The database configuration should be specified using environment variables while starting the GitLab CI image.

Before you start the GitLab CI image create user and database for GitLab CI.

```bash
mysql -uroot -p
CREATE USER 'gitlab_ci'@'%.%.%.%' IDENTIFIED BY 'password';
CREATE DATABASE IF NOT EXISTS `gitlab_ci_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `gitlab_ci_production`.* TO 'gitlab_ci'@'%.%.%.%';
```

To make sure the database is initialized start the container with **app:rake db:setup** option.

**NOTE: This should be done only for the first run**.

*Assuming that the mysql server host is 192.168.1.100*

```bash
docker run -name gitlab-ci -i -t -rm \
  -e "GITLAB_URL=http://172.17.0.2" \
  -e "DB_HOST=192.168.1.100" -e "DB_NAME=gitlab_ci_production" \
  -e "DB_USER=gitlab_ci" -e "DB_PASS=password" \
  sameersbn/gitlab-ci app:rake db:setup
```

This will initialize the GitLab CI database. Now that the database is initialized, start the container normally.

```bash
docker run -name gitlab-ci -d \
  -e "GITLAB_URL=http://172.17.0.2" \
  -e "DB_HOST=192.168.1.100" -e "DB_NAME=gitlab_ci_production" \
  -e "DB_USER=gitlab_ci" -e "DB_PASS=password" \
  sameersbn/gitlab-ci
```

### PostgreSQL

#### External PostgreSQL Server
The image also supports using an external PostgreSQL Server. This is also controlled via environment variables.

```bash
createuser gitlab_ci
createdb -O gitlab_ci gitlab_ci_production
```

To make sure the database is initialized start the container with **app:rake db:setup** option.

**NOTE: This should be done only for the first run**.

*Assuming that the PostgreSQL server host is 192.168.1.100*

```bash
docker run -name gitlab-ci -i -t -rm \
  -e "GITLAB_URL=http://172.17.0.2" \
  -e "DB_TYPE=postgres" -e "DB_HOST=192.168.1.100" \
  -e "DB_NAME=gitlab_ci_production" \
  -e "DB_USER=gitlab_ci" -e "DB_PASS=password" \
  sameersbn/gitlab-ci app:rake db:setup
```

This will initialize the GitLab CI database. Now that the database is initialized, start the container normally.

```bash
docker run -name gitlab-ci -d \
  -e "GITLAB_URL=http://172.17.0.2" \
  -e "DB_TYPE=postgres" -e "DB_HOST=192.168.1.100" \
  -e "DB_NAME=gitlab_ci_production" \
  -e "DB_USER=gitlab_ci" -e "DB_PASS=password" \
  sameersbn/gitlab-ci
```

### Mail
The mail configuration should be specified using environment variables while starting the GitLab CI image. The configuration defaults to using gmail to send emails and requires the specification of a valid username and password to login to the gmail servers.

The following environment variables need to be specified to get mail support to work.

* SMTP_DOMAIN (defaults to www.gmail.com)
* SMTP_HOST (defaults to smtp.gmail.com)
* SMTP_PORT (defaults to 587)
* SMTP_USER
* SMTP_PASS
* SMTP_STARTTLS (defaults to true)

```bash
docker run -name gitlab-ci -d \
  -e "SMTP_USER=USER@gmail.com" -e "SMTP_PASS=PASSWORD" \
  sameersbn/gitlab-ci
```

If you are not using google mail, then please configure the  SMTP host and port using the SMTP_HOST and SMTP_PORT configuration parameters.

__NOTE:__

I have only tested standard gmail and google apps login. I expect that the currently provided configuration parameters should be sufficient for most users. Please look up the [Available Configuration Parameters](#available-configuration-parameters) section for all available SMTP configuration options.

### Putting it all together

```bash
docker run -name gitlab-ci -d -h gitlab-ci.local.host \
  -v /opt/gitlab-ci/mysql:/var/lib/mysql \
  -e "GITLAB_URL=http://172.17.0.2" \
  -e "GITLAB_CI_HOST=gitlab-ci.local.host" \
  -e "GITLAB_CI_EMAIL=gitlab@local.host" \
  -e "GITLAB_CI_SUPPORT=support@local.host" \
  -e "SMTP_USER=USER@gmail.com" -e "SMTP_PASS=PASSWORD" \
  sameersbn/gitlab-ci
```

If you are using an external mysql database

```bash
docker run -name gitlab-ci -d -h gitlab-ci.local.host \
  -e "DB_HOST=192.168.1.100" -e "DB_NAME=gitlab_ci_production" \
  -e "DB_USER=gitlab_ci" -e "DB_PASS=password" \
  -e "GITLAB_URL=http://172.17.0.2" \
  -e "GITLAB_CI_HOST=gitlab-ci.local.host" \
  -e "GITLAB_CI_EMAIL=gitlab@local.host" \
  -e "GITLAB_CI_SUPPORT=support@local.host" \
  -e "SMTP_USER=USER@gmail.com" -e "SMTP_PASS=PASSWORD" \
  sameersbn/gitlab-ci
```

### Available Configuration Parameters

Below is the complete list of available options that can be used to customize your GitLab CI installation.

- **GITLAB_URL**: Url of the GitLab server to allow connections from. No defaults. Automatically configured when a GitLab server is linked using docker links feature.
- **GITLAB_CI_HOST**: The hostname of the GitLab CI server. Defaults to localhost.
- **GITLAB_CI_PORT**: The port number of the GitLab CI server. Defaults to 80.
- **GITLAB_CI_EMAIL**: The email address for the GitLab CI server. Defaults to gitlab@localhost.
- **GITLAB_CI_SUPPORT**: The support email address for the GitLab CI server. Defaults to support@localhost.
- **REDIS_HOST**: The hostname of the redis server. Defaults to localhost
- **REDIS_PORT**: The connection port of the redis server. Defaults to 6379.
- **UNICORN_WORKERS**: The number of unicorn workers to start. Defaults to 2.
- **UNICORN_TIMEOUT**: Sets the timeout of unicorn worker processes. Defaults to 60 seconds.
- **DB_TYPE**: The database type. Possible values: mysql, postgres. Defaults to mysql.
- **DB_HOST**: The database server hostname. Defaults to localhost.
- **DB_PORT**: The database server port. Defaults to 3306 for mysql and 5432 for postgresql.
- **DB_NAME**: The database database name. Defaults to gitlab_ci_production
- **DB_USER**: The database database user. Defaults to root
- **DB_PASS**: The database database password. Defaults to no password
- **DB_POOL**: The database database connection pool count. Defaults to 10.
- **SMTP_DOMAIN**: SMTP domain. Defaults to www.gmail.com
- **SMTP_HOST**: SMTP server host. Defaults to smtp.gmail.com.
- **SMTP_PORT**: SMTP server port. Defaults to 587.
- **SMTP_USER**: SMTP username.
- **SMTP_PASS**: SMTP password.
- **SMTP_STARTTLS**: Enable STARTTLS. Defaults to true.

# Maintenance

## SSH Login
There are two methods to gain root login to the container, the first method is to add your public rsa key to the authorized_keys file and build the image.

The second method is use the dynamically generated password. Every time the container is started a random password is generated using the pwgen tool and assigned to the root user. This password can be fetched from the docker logs.

```bash
docker logs gitlab-ci 2>&1 | grep '^User: ' | tail -n1
```

This password is not persistent and changes every time the image is executed.

# Upgrading

To upgrade to newer GitLab CI releases, simply follow this 4 step upgrade procedure.

- **Step 1**: Stop the currently running image

```bash
docker stop gitlab-ci
```

- **Step 2**: Update the docker image.

```bash
docker pull sameersbn/gitlab-ci
```

- **Step 3**: Migrate the database.

```bash
docker run -name gitlab-ci -i -t -rm [OPTIONS] \
  sameersbn/gitlab-ci app:rake db:migrate
```

- **Step 4**: Start the image

```bash
docker run -name gitlab-ci -d [OPTIONS] sameersbn/gitlab-ci
```


## References
  * https://www.gitlab.com/gitlab-ci/
  * https://gitlab.com/gitlab-org/gitlab-ci/blob/master/README.md