---
title: "Secure Self-Hosted Ollama Server"
date: 2024-10-26T19:22:03+00:00
lastmod: 2024-10-26T19:22:03+00:00
# weight: 1
tags: ["Ollama", "Docker", "Caddy"]
categories: ["Infrastructure"]
authors: ["Panos Korovesis"]
draft: false
summary: "Deploy a secure ollama server using docker and Caddy for authentication"
---

## Introduction 🔎

The goal of this post is to create a self-hosted ollama server, used for deploying and testing open source LLMs locally.

The main issue is that by default, there is no password (or any other) protection for the ollama server. This means that if we were to deploy this in our server and then expose it to the internet, in order to access if from other machines, _we would have an issue_ as anyone could use it _simply with the url_

To solve this we are going to see how we can set-up a dockerized ollama server with Caddy authentication for one user.

{{< alert icon="github" >}}
The entire project can be found in my [github](https://github.com/panoskorovesis/secure-ollama-server)
{{< /alert >}}

## Deployment ✈

### Starting the Ollama Server

__Deploy__ our server we simply have to use the following command:

```bash
docker compose -f secure-ollama-server/docker-compose.yaml --env-file secure-ollama-server/.env up -d
```

{{< alert >}}
In case the machine we want to run ollama on has a gpu, we have to also use the docker-compose.gpu.yaml file. The command can be seen bellow
{{< /alert >}}

```
docker compose -f secure-ollama-server/docker-compose.yaml -f secure-ollama-server/docker-compose.gpu.yaml --env-file secure-ollama-server/.env up -d
```

{{< alert icon="docker" cardColor=#5667f4 textColor="black">}}
Docker will take care of the rest
{{< /alert >}}

### Stopping the Ollama Server

To __stop the server__ after we are finished using it, we have to use the following command:

```bash
docker compose -f secure-ollama-server/docker-compose.yaml --env-file secure-ollama-server/.env down
```

## How to use ❔

In order to use the server, we have to _add the username and the password we specified in the .env file in our request_.

An example in curl can be seen bellow:

```bash
curl -u user:password 127.0.0.1:8200/api/generate -d '{"model" : "phi", "prompt" : "Why is the sky blue?", "stream" : false}'
```

{{< alert >}}
Using an __incorrect__ password or user will result in an __empty request response__.
{{< /alert >}}

## Development 🛠

The following sections contain technical information about the implementation.

### Enviromental Variables

In order for the Caddy Authentication to work, we need to __set up some enviromental variables__.

Those must be placed in a `.env` file, located in the same folder as the `docker-compose.yaml`

We can easily create this file by taking a look at the `.env-template` file. An example of our `.env` file is the following:

```bash
CADDY_USERNAME=admin
CADDY_PASSWORD=secret-password
```

### Docker

Now, let's take a look at the `docker-compose.yaml` file:

```yaml
version: '3.8'

services:
  ollama:
    container_name: ollama-server
    build:
      context: .
      dockerfile: Dockerfile
    pull_policy: always
    tty: true
    restart: always
    ports:
      - 8200:80
    volumes:
      - ./ollama:/root/.ollama
    environment:
      - CADDY_USERNAME=${CADDY_USERNAME}
      - CADDY_PASSWORD=${CADDY_PASSWORD}
```

1. The ollama server is deployed on __port 8200__.
2. There is a __restart-always__ policy placed on the container.
3. A __volumne__ is required for persistence.
4. The volume is created in the __same folder__ as the docker-compose.yaml.
5. __Caddy enviromental__ variables are needed to set up the user and it's password.

### Caddy Setup

Caddy is __automatically configured for us__ by the `Dockerfile`. Let's take a look to see how this is done.

```Dockerfile
FROM ollama/ollama:latest

# Update and install wget to download caddy
RUN apt-get update && apt-get install -y wget

# Download and install caddy
RUN wget --no-check-certificate https://github.com/caddyserver/caddy/releases/download/v2.7.6/caddy_2.7.6_linux_amd64.tar.gz \
    && tar -xvf caddy_2.7.6_linux_amd64.tar.gz \
    && mv caddy /usr/bin/ \
    && chown root:root /usr/bin/caddy \
    && chmod 755 /usr/bin/caddy

# Copy the Caddyfile to the container
COPY Caddyfile /etc/caddy/Caddyfile

# Set the environment variable for the ollama host
ENV OLLAMA_HOST 0.0.0.0

# Expose the port that caddy will listen on
EXPOSE 80

# Copy a script to start both ollama and caddy
COPY start_server.sh /start_server.sh
RUN chmod +x /start_server.sh

# Set the entrypoint to the script
ENTRYPOINT ["/start_server.sh"]
```

1. We __download the v2.7.6 version__ of the Caddy Server and install it.
2. The appropriate permissions and onwerships are applied.
3. The `start_server.sh` script is triggered at the end.

The `start_server.sh` performs the following actions:

1. Checks for the presence of the required _Enviromental Variables_
2. Starts the Ollama server.
3. Starts the Caddy server.
4. Handles shutdown signals, in order for the container to exit grasefully.