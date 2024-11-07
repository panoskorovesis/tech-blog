---
title: "Secure Self-Hosted Ollama Server"
date: 2024-10-26T19:22:03+00:00
lastmod: 2024-10-26T19:22:03+00:00
# weight: 1
tags: ["Ollama", "Docker"]
categories: ["Infrastructure"]
authors: ["Panos Korovesis"]
draft: false
summary: "Deploy a secure ollama server using docker and Caddy for authentication"
---

## Introduction

The goal of this post is to create a self-hosted ollama server, used for deploying and testing open source LLMs locally.

The main issue is that by default, there is no password (or any other) protection for the ollama server. This means that if we were to deploy this in our server and then expose it to the internet, in order to access if from other machines, _we would have an issue_ as anyone could use it _simply with the url_

To solve this we are going to see how we can set-up a dockerized ollama server with Caddy authentication for one user.