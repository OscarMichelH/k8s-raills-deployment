# k8s-raills-deployment

This repository contains reference files to allow us deploy a Rails application in a k8s cluster using docker/compose.

## Requirements
- Upload your image to docker hub or other provider to pull and mount image in a deployment resource inside our cluster



To mount necessary resources for rails we need rails itself, a database

- dockerfile:
We are specifying ruby image use to mount, ENV vars required to manage rails execution, and other rails dependencies like postgresql, node, etc.
```
FROM ruby:3.1.2-alpine

ENV APP_PATH /var/app
ENV BUNDLE_VERSION 2.3.25
ENV BUNDLE_PATH /usr/local/bundle/gems
ENV TMP_PATH /tmp/
ENV RAILS_LOG_TO_STDOUT true
ENV RAILS_PORT 3000

COPY ./dev-docker.entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

#setup deps for rails
RUN apk -U add --no-cache \
build-base \
git \
postgresql-dev \
postgresql-client \
libxml2-dev \
libxslt-dev \
nodejs \
yarn \
imagemagick \
tzdata \
less \
&& rm -rf /var/cahe/apk/* \
&& mkdir -p $APP_PATH

RUN gem install bundler --version "$BUNDLE_VERSION" \
    && rm  -rf $GEM_HOME/cache/*

WORKDIR $APP_PATH
COPY . $APP_PATH
COPY Gemfile Gemfile.lock ./
EXPOSE $RAILS_PORT
ENTRYPOINT ["bundle", "exec"]
```

- docker-compose.yml: We are creating the volume to allocate our db, and db service that will be used by rail, finally app creation from previous Dockerfile connecting volumes, setting the port and commands to be executed after creation (entrypoint.sh)
```
version: "3"

volumes:
  db-data:
    external: false
  gem_cache:
  shared_data:

services:
  db:
    image: postgres:15.1-alpine
    env_file:
      - config/application.yml
    volumes:
      - db-data:/usr/local/pgsql/data

  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: balansee
    volumes:
      - .:/var/app
      - shared_data:/var/shared
      - gem_cache:/usr/local/bundle/gems
    ports:
      - "3000:3000"
    depends_on:
      - db
    stdin_open: true
    tty: true
    env_file:
      - config/application.yml
    entrypoint: entrypoint.sh
    command: ['rails', 's', '-p', '3000', '-b', '0.0.0.0']
```
Note: Because I'm using figaro gem to handle ENV vars, I'm setting directly config/application.yml as secrets file, but feel free to put environments directly inside docker-compose file or use the next example as reference:
config/application.yml
```
RACK_ENV: development
RAILS_ENV: development
RAILS_LOG_TO_STDOUT: "true"
RAILS_SERVE_STATIC_FILES: "true"
SECRET_KEY_BASE: "YOUR_SECRET"
POSTGRES_DB: "YOUR DB"
DATABASE_USER: "postgres"
DATABASE_PASSWORD: "root"
DATABASE_URL: postgres://postgres:root@db:5432/YOUR_DB
POSTGRES_HOST_AUTH_METHOD: "trust"
DATABASE_PORT: "5432"
SMTP_USERNAME: "YOUR_SMTP_USER"
SMTP_PASSWORD: "YOUR_SMTP_PASS"
SMTP_SERVER: YOUR_SMTP_SERVICE
```

- entrypoint.sh: Commands to execute rails app, first we print the current environment, checks any missing dependency for gemfile, remove any existing rails execution, and execute any possible exec
```
#!/bin/sh

set -e

echo "ENVIRONMENT: $RAILS_ENV"

# check bundle
bundle check || bundle install

# remove any existing PID
rm -f  $APP_PATH/tmp/pids/server.pid

# run anythin by prepending bundle exec to the passed command
bundle exec ${@}
```

This first part is only to comprobe if we have everything ok, so we gonna proceed to test our app running in docker
`docker-compose up`

If everything is ok, and your app is running correcly in localhost:3000, then we can upload our image to docker using the command:
```
docker buildx build --platform linux/amd64,linux/arm64 -t YOUR_DOCKER_USERNAME/YOUR_IMAGE_NAME:latest --push .
```
I'm using buildx to include arm64 arquitectures and avoid compatibility issues if I want run my app in a M1 computer for example.

Finally use .k8s files as reference to create resources in any k8s cluster, basically we gonna create a persistence volume to allocate database, and a deployment resourcer that will use our previous rails app docker image, apply changes using command `kubectl apply -f .k8s` 