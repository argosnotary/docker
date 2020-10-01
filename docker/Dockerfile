#
# Copyright (C) 2019 - 2020 Rabobank Nederland
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#         http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

ARG ARGOS_FRONTEND_VERSION
ARG ARGOS_SERVICE_VERSION
ARG ARGOS_FRONTEND_IMAGE
ARG ARGOS_SERVICE_IMAGE

FROM argosnotary/${ARGOS_FRONTEND_IMAGE}:${ARGOS_FRONTEND_VERSION} as frontend

FROM argosnotary/${ARGOS_SERVICE_IMAGE}:${ARGOS_SERVICE_VERSION} as argos-service

FROM argosnotary/oauth-stub:2.25.1 as oauth-stub

FROM phusion/baseimage:0.11 as run-server-stage

RUN mkdir /usr/lib/argos

######################################################################
# Begin install mongo
######################################################################

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN addgroup --system nginx \
    && adduser --system --disabled-login --ingroup nginx --no-create-home --home /nonexistent --gecos "nginx user" --shell /bin/false nginx \
    && groupadd -r mongodb && useradd -r -g mongodb mongodb \
    && groupadd -r argos && useradd -r -g argos argos \
    && groupadd -r oauth && useradd -r -g oauth oauth

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		ca-certificates \
		jq \
		numactl \
		gettext \
		wget \
	; \
	if ! command -v ps > /dev/null; then \
		apt-get install -y --no-install-recommends procps; \
	fi; \
	rm -rf /var/lib/apt/lists/*
	
RUN set -ex; \
  \
  apt-get update; \
  apt-get install -y --no-install-recommends gnupg dirmngr

# grab gosu for easy step-down from root (https://github.com/tianon/gosu/releases)
ENV GOSU_VERSION 1.11
# grab "js-yaml" for parsing mongod's YAML config files (https://github.com/nodeca/js-yaml/releases)
ENV JSYAML_VERSION 3.13.0

RUN set -ex; \
	dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \
	wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch"; \
	chmod +x /usr/local/bin/gosu; \
	gosu --version; \
	gosu nobody true; \
	\
	wget -O /js-yaml.js "https://github.com/nodeca/js-yaml/raw/${JSYAML_VERSION}/dist/js-yaml.js"

RUN mkdir /docker-entrypoint-initdb.d

RUN set -ex; \
	export GNUPGHOME="$(mktemp -d)"; \
	for key in $GPG_KEYS; do \
		gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
	done; \
	gpg --batch --export $GPG_KEYS > /etc/apt/trusted.gpg.d/mongodb.gpg; \
	command -v gpgconf && gpgconf --kill all || :; \
	rm -r "$GNUPGHOME"; \
	apt-key list

# Allow build-time overrides (eg. to build image with MongoDB Enterprise version)
# Options for MONGO_PACKAGE: mongodb-org OR mongodb-enterprise
# Options for MONGO_REPO: repo.mongodb.org OR repo.mongodb.com
# Example: docker build --build-arg MONGO_PACKAGE=mongodb-enterprise --build-arg MONGO_REPO=repo.mongodb.com .
ARG MONGO_PACKAGE=mongodb-org
ARG MONGO_REPO=repo.mongodb.org
ENV MONGO_PACKAGE=${MONGO_PACKAGE} MONGO_REPO=${MONGO_REPO}

ENV MONGO_MAJOR 4.2
ENV MONGO_VERSION 4.2.8
# bashbrew-architectures:amd64 arm64v8 s390x
RUN echo "deb http://$MONGO_REPO/apt/ubuntu bionic/${MONGO_PACKAGE%-unstable}/$MONGO_MAJOR multiverse" | tee "/etc/apt/sources.list.d/${MONGO_PACKAGE%-unstable}.list"

RUN wget -qO - https://www.mongodb.org/static/pgp/server-4.2.asc | apt-key add - \
    && echo "deb http://repo.mongodb.org/apt/debian buster/mongodb-org/4.2 main" | tee /etc/apt/sources.list.d/mongodb-org-4.2.list

RUN set -x \
# installing "mongodb-enterprise" pulls in "tzdata" which prompts for input
	&& export DEBIAN_FRONTEND=noninteractive \
	&& apt-get update \
	&& apt-get install -y \
		${MONGO_PACKAGE}=$MONGO_VERSION \
		${MONGO_PACKAGE}-server=$MONGO_VERSION \
		${MONGO_PACKAGE}-shell=$MONGO_VERSION \
		${MONGO_PACKAGE}-mongos=$MONGO_VERSION \
		${MONGO_PACKAGE}-tools=$MONGO_VERSION \
	&& rm -rf /var/lib/apt/lists/* \
	&& rm -rf /var/lib/mongodb \
	&& mv /etc/mongod.conf /etc/mongod.conf.orig

RUN mkdir -p /data/db /data/configdb \
	&& chown -R mongodb:mongodb /data/db /data/configdb
	
VOLUME /data/db /data/configdb

EXPOSE 27017
######################################################################
# End install mongo
######################################################################

######################################################################
# Begin install java
######################################################################

ENV JAVA_VERSION   11.0.6
#ENV JAVA_UBUNTU_VERSION   8u191-b12-2ubuntu0.16.04.1

# see https://bugs.debian.org/775775
# and https://github.com/docker-library/java/issues/19#issuecomment-70546872
ENV CA_CERTIFICATES_JAVA_VERSION 20160321ubuntu1

######################################################################
# End install java
######################################################################

######################################################################
# Begin install oauth stub
######################################################################

ENV WIREMOCK_VERSION 2.26.3

# grab wiremock standalone jar
RUN mkdir -p /var/wiremock/lib/ /usr/lib/argos/oauth/mappings \
  && wget https://repo1.maven.org/maven2/com/github/tomakehurst/wiremock-standalone/$WIREMOCK_VERSION/wiremock-standalone-$WIREMOCK_VERSION.jar \
    -O /var/wiremock/lib/wiremock-standalone.jar

COPY --from=oauth-stub /home/wiremock/mappings /usr/lib/argos/oauth/mappings
COPY --from=oauth-stub /home/wiremock/__files /usr/lib/argos/oauth/__files

######################################################################
# End install oauth stub
######################################################################

RUN set -x \
	&& apt-get update \
	&& apt-get install -y \
		openjdk-11-jre-headless \
		nginx

COPY --from=frontend /usr/share/nginx/html/ /usr/share/nginx/html
COPY --from=frontend /etc/nginx/nginx.conf /etc/nginx/nginx.conf
COPY --from=frontend /image_config/nginx.conf.template /

COPY --from=argos-service /argos.jar /usr/lib/argos/argos-service.jar

ENV BACKEND_URL=http://localhost:8080
      
ENV jwt.token.secret           Z2LcVpgCuGtpb5rBDNIIWGe8RXqYGWDpWFjWhYQUS2Z8rjE2/GTJlu0s6BRYG0Mn0kV8StC9cFE83Jkl68RCkw==
ENV server.use-forward-headers 'true'

# Add scripts for starting runit services
COPY resources /tmp/resources
RUN  cp -R /tmp/resources/service/* /etc/service/ && chmod +x /etc/service/*/run

COPY mongo_entry.sh /usr/lib/argos/
RUN echo "rs.initiate({'_id':'rs0','members':[{'_id':0,'host':'127.0.0.1:27017'}]});" > /docker-entrypoint-initdb.d/replica-init.js

RUN chown -R nginx:nginx /var/log/nginx /var/lib/nginx \
    && chmod -R 755 /usr/lib/argos
    
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 80 8080 8087
