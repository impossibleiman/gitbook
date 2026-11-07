---
description: >-
  This page explains the structure and process of how I setup a Discord Bot and
  its own API for a dashboard.
---

# Discord Bot & Bot Website

#### 30th July 2026

My second year group project was to pick a topic from a set list.\
The topic that stuck out to me most was `Discord/Twitch Bot` as at the time I was the chair for the Minecraft Society at MMU which heavily relied on Discord for its communication.

We created a brilliant Minimum Viable Product (MVP) for a Discord Bot followed by a working dashboard which used Discord's OAuth2, connecting to servers where the user has admin.



#### 3rd August 2026

I want to further improve on this as a solution for The Union. The Discord Bot's current dashboard was tailored to the Minecraft Society, however I now want to be able to portray the bot as the unified bot for all societies in The Union. The goal of this side project is to be able to publish something as my own work to The Union which could run for as little as my remaining time at university or for as long as the code is maintained.

## VM 101

To start I will configure my VM 101 to add a new Docker Network:

```shellscript
docker network create discord-internal
```

Confirming it is now present with&#x20;

```shellscript
docker network ls
```

Entering my discord-bot folder, I need to create the docker-compose.yml and .env:

{% code title="docker-compose.yml" expandable="true" %}
```docker
services:
  the-union-bot:
    container_name: the-union-bot
    build: .
    restart: unless-stopped

    mem_limit: 3g
    cpus: 2

    dns:
      - 192.168.0.3
      - 1.1.1.1

    environment:
      TZ: Europe/London

    volumes:
      - ./.env:/app/.env
      - ./data:/app/data

    networks:
      - discord-internal

networks:
  discord-internal:
    external: true
```
{% endcode %}

{% code title=".env" expandable="true" %}
```dotenv
DISCORD_TOKEN=
DISCORD_CLIENT_ID=
DISCORD_CLIENT_SECRET=
```
{% endcode %}

Running the docker container was successful after troubleshooting an oversight of initially making the network internal (meaning it can't reach Discord's gateway). As I'm using a partial solution, features like /cat (shows you a random picture from r/cat) work from the get go. We now need to connect this to a dashboard.&#x20;
