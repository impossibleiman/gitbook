---
description: It's been a while
---

# HomeAssistant Monitoring

#### 7th November 2026 @ 6:47AM

It has been a couple months of using my home server and so far it's been very useful. Combined with tailscale I have been able to browse my own media, whether its my large library of photos and videos. I've even finished off Game of Thrones, a show I started watching about 4 years ago but never made it past season 2!

I have made some unnoted changes since I last wrote such as using Caddy's certificate for HTTPS. I may go back to write about this at some point but for now I want to setup my Home Assistant container



## HomeAssistant's responsibilities?

Whilst I could do everything manually through each container's app or a terminal, I want to simplify the actions and analytics to a single page (plus I've heard it's good to show off to an interviewer).

Here are some of the things we'll look to make it do:

* Starting, stopping and restarting Docker containers.
* Monitoring Docker health and resource usage.
* Checking for image updates.
* Applying updates when I authorise them.
* Monitoring external services and infrastructure.
* Sending notifications when something fails.
* Controlling AMP game instances.
* Running scheduled maintenance tasks.

Not all of these may be possible but we will reflect that at the end.

## HomeAssistant Dashboard & prerequisites

I had only briefly used Home Assistant to make a script to wake my PC up via LAN so I could remotely connect to it via moonlight & sunshine. Now I need to add a new dashboard from the settings page and set it as the default. This will replace the standard "overview" page.

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

Editing this page, I have added 3 sections for now: System Status; Docker Containers and Infrastructure. We can add more later if we need to.

## Connecting Docker to HomeAssistant

We can use a Docker Socket Proxy to connect the two together safely without giving HomeAssistant root access. I used to use watchtower to continuously update one of my containers this way but I had forgotten to keep it updated which stopped it from working. Sounds like kind of a counter-intuitive idea back then...&#x20;



The Docker Socket Proxy will act as a Read-only API giving Docker container information for the time being via the Docker host's loopback interface. Once we have fully set it up then we can upgrade the system to act as a control point.



{% code title="docker-compose.yml" expandable="true" %}
```docker
services:
  docker-socket-proxy:
    image: ghcr.io/tecnativa/docker-socket-proxy:latest
    container_name: docker-socket-proxy
    restart: unless-stopped

    environment:
      CONTAINERS: "1"
      EVENTS: "1"
      INFO: "1"
      IMAGES: "1"
      NETWORKS: "1"
      PING: "1"
      VERSION: "1"

      POST: "0"
      AUTH: "0"
      SECRETS: "0"
      EXEC: "0"
      VOLUMES: "0"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

    ports:
      - "127.0.0.1:2375:2375"
```
{% endcode %}

To confirm it works after running

{% code title="" expandable="true" %}
```shellscript
curl http://127.0.0.1:2375/_ping
```
{% endcode %}

outputting OK

We can start with monitoring container states before moving onto CPU, memory and other hardware information.&#x20;

{% code title="/opt/docker/homeassistant/config/configuration.yaml" expandable="true" %}
```yaml

# ----------------------------------------------------
# Stuff that was already in here before
# ----------------------------------------------------

# Loads default set of integrations. Do not remove.
default_config:

# Load frontend themes from the themes folder
frontend:
  themes: !include_dir_merge_named themes

automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml

http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.20.0.0/16

wake_on_lan:

switch:
  - platform: wake_on_lan
    name: Gaming PC
    mac: "9C:6B:00:17:76:DD"
    host: "192.168.0.200"
    broadcast_address: "192.168.0.255"
    broadcast_port: 9
    
    
# ---------------------
# New code


rest:
  - resource: http://127.0.0.1:2375/containers/json?all=1
    method: GET
    scan_interval: 30
    sensor:
      - name: "Docker Containers"
        unique_id: docker_containers
        value_template: "{{ value_json | count }}"

      - name: "Docker Running Containers"
        unique_id: docker_running_containers
        value_template: "{{ value_json | selectattr('State', 'equalto', 'running') | list | count }}"

      - name: "Docker Stopped Containers"
        unique_id: docker_stopped_containers
        value_template: "{{ value_json | rejectattr('State', 'equalto', 'running') | list | count }}"

```
{% endcode %}

then:

{% code title="" expandable="true" %}
```shellscript
docker restart homeassistant
```
{% endcode %}

Now visiting HomeAssistant's Settings > Developer Tools > States (along the top) and filtering entities for "docker\_" will show us the 3 added sensors with their states/values:&#x20;

* sensor.docker\_containers: 24 (total containers)&#x20;
* sensor.docker\_running\_containers: 20
* sensor.docker\_stopped\_containers: 4&#x20;

These values will of course be different for someone else's home server.

These can now be added to the dashboard. Under the Docker Container card:

* Click the plus&#x20;
* Select the "By card" header
* Search and find the card labelled "Entities"

<figure><img src="../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

Given the context of the card's name "Docker Containers" you can edit each row, simplifying the labels

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

This is a very very simple method of monitoring our containers. We can take this a step further by showing a live inventory of containers, their statuses, their image, and restart status.



{% code title="/opt/docker/homeassistant/config/docker_inventory.py" expandable="true" %}
```python
import json
import urllib.request
from datetime import datetime, timezone

url = "http://127.0.0.1:2375/containers/json?all=1"

with urllib.request.urlopen(url, timeout=10) as response:
    docker_data = json.load(response)

containers = []

for container in docker_data:
    containers.append({
        "name": container["Names"][0].lstrip("/"),
        "image": container["Image"],
        "state": container["State"],
        "status": container["Status"],
        "created": container["Created"],
        "created_date": datetime.fromtimestamp(
            container["Created"], timezone.utc
        ).strftime("%d/%m/%Y %H:%M UTC")
    })

containers.sort(key=lambda item: item["created"], reverse=True)

print(json.dumps({"containers": containers}))
```
{% endcode %}

This script will help query the docker API, retrieve all containers and in alphabetical order of names return their name, image, state and status.

We can append the config for homeassistant to add this as a sensor

{% code title="/opt/docker/homeassistant/config/configuration.yaml" expandable="true" %}
```yaml
# ... existing code from before

command_line:
  - sensor:
      name: Docker Container Inventory
      unique_id: docker_container_inventory
      command: "python3 /config/docker_inventory.py"
      command_timeout: 15
      scan_interval: 30
      value_template: "{{ value_json.containers | count }}"
      json_attributes:
        - containers
```
{% endcode %}

Using a markdown card in the dashboard we can display all our containers through the code editor

{% code title="" expandable="true" %}
```yaml
type: markdown
title: Docker Container Inventory
content: >
  **{{ states('sensor.docker_container_inventory') }} containers**


  | Container | State | Created | Image |

  |:---|:---:|:---|:---|

  {% set containers = state_attr('sensor.docker_container_inventory',
  'containers') or [] %}

  {% for c in containers %}

  | `{{ c.name }}` | {{ c.state }} | {{ c.created_date }} | `{{ c.image }}` |

  {% endfor %}
grid_options:
  columns: 24
  rows: auto

```
{% endcode %}

<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

For the purpose of the image, I have resized the card. It might look unfinished for now but at least we know it works! We can also remove the first card now

Now I want to add 3 things

* Clicking on the name of the container takes you to the service
* Clicking on the state of the container takes you to its logs
* Colour coding on the states

For the logs we can use something called Dozzle which lets you follow logs in real time.

To keep things secure, we'll need a second proxy for Dozzle as CONTAINERS=1 (Socket Proxy connection) also permits GET access to read container filesystems, export filesystems, and inspect process arguments.

{% code title="/opt/docker/dozzle-proxy" expandable="true" %}
```docker
services:
  dozzle-socket-proxy:
    image: lscr.io/linuxserver/socket-proxy:latest
    container_name: dozzle-socket-proxy
    restart: unless-stopped

    environment:
      CONTAINERS: "1"
      EVENTS: "1"
      INFO: "1"
      PING: "1"
      VERSION: "1"

      ALLOW_LOGS: "1"
      ALLOW_ARCHIVE: "0"
      ALLOW_CHANGES: "0"
      ALLOW_EXPORT: "0"
      ALLOW_TOP: "0"
      ALLOW_START: "0"
      ALLOW_STOP: "0"
      ALLOW_RESTARTS: "0"
      ALLOW_PAUSE: "0"
      ALLOW_UNPAUSE: "0"

      POST: "0"
      AUTH: "0"
      EXEC: "0"
      IMAGES: "0"
      NETWORKS: "0"
      VOLUMES: "0"
      SECRETS: "0"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

    group_add:
      - "986"

    read_only: true

    tmpfs:
      - /run

    networks:
      - dozzle-backend

networks:
  dozzle-backend:
    name: dozzle-backend
    driver: bridge
```
{% endcode %}

This proxy has no published ports

Once this proxy is running, we just need to check two things

* Can it list containers

{% code title="" expandable="true" %}
```shellscript
docker exec dozzle-socket-proxy sh -c 'wget -qO- http://127.0.0.1:2375/_ping'
```
{% endcode %}

it responds with `OK` meaning it's all good

* Does it have the right permissions?

Expecting Success:

{% code title="Container Discovery - should succeed" expandable="true" %}
```shellscript
docker exec dozzle-socket-proxy sh -c 'wget -qO- "http://127.0.0.1:2375/containers/json?all=1" | head -c 500'
```
{% endcode %}

{% code title="Container Logs - should succeed" expandable="true" %}
```shellscript
docker exec dozzle-socket-proxy sh -c 'wget -qO- "http://127.0.0.1:2375/containers/homeassistant/logs?stdout=1&stderr=1&tail=5"'
```
{% endcode %}

Expecting Denial:

{% code title="Filesystems Access - should NOT succeed" expandable="true" %}
```shellscript
docker exec dozzle-socket-proxy sh -c 'wget -S -O- "http://127.0.0.1:2375/containers/homeassistant/archive?path=/etc/passwd" 2>&1'
```
{% endcode %}

{% code title="Process Inspection - should NOT succeed" expandable="true" %}
```shellscript
docker exec dozzle-socket-proxy sh -c 'wget -S -O- "http://127.0.0.1:2375/containers/homeassistant/top" 2>&1'
```
{% endcode %}



Now we can create a Dozzle login

{% code title="" expandable="true" %}
```shellscript
mkdir -p /opt/docker/dozzle/data
```
{% endcode %}

{% code title="/opt/docker/dozzle" expandable="true" %}
```shellscript
cd /opt/docker/dozzle

docker run --rm amir20/dozzle:latest generate admin --password 'YOUR_NEW_PASSWORD' > ./data/users.yml
```
{% endcode %}

this password may be stored in your shell history so you can alternatively use&#x20;

{% code title="" expandable="true" %}
```shellscript
read -rsp "New Dozzle password: " DOZZLE_PASSWORD
echo
docker run --rm amir20/dozzle:latest generate admin --password "$DOZZLE_PASSWORD" > ./data/users.yml
unset DOZZLE_PASSWORD
```
{% endcode %}

{% code title="docker-compose.yml" expandable="true" %}
```docker
services:
  dozzle:
    image: amir20/dozzle:latest
    container_name: dozzle
    restart: unless-stopped

    environment:
      DOZZLE_REMOTE_HOST: tcp://dozzle-socket-proxy:2375
      DOZZLE_AUTH_PROVIDER: simple

    volumes:
      - ./data:/data

    networks:
      - proxy
      - dozzle-backend

networks:
  proxy:
    external: true

  dozzle-backend:
    external: true
```
{% endcode %}

{% code title="Caddyfile" expandable="true" %}
```
...

dockerlogs.imanizadyar.com {
    tls internal
    reverse_proxy dozzle:8080
}
```
{% endcode %}

Then do your usual&#x20;

{% code title="" expandable="true" %}
```
docker exec caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```
{% endcode %}

and if valid:

{% code title="" expandable="true" %}
```
docker exec caddy caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
```
{% endcode %}

Then add your dns to your local dns records via pi-hole. Mine is dockerlogs.imanizadyar.com @ 192.168.0.50

Now using your admin login you setup, login to Dozzle at your newly created local DNS url.

<figure><img src="../.gitbook/assets/image (8).png" alt="" width="132"><figcaption></figcaption></figure>

and now we can make the changes for the dashboard, replacing your old markdown card code:

{% code title="Markdown card configuration" expandable="true" %}
```yaml

type: markdown
content: |
  **{{ states('sensor.docker_container_inventory') }} containers**

  | Container | State | Created | Image |
  |:---|:---:|:---|:---|
  {% set containers = state_attr('sensor.docker_container_inventory', 'containers') or [] %}
  {% set service_urls = {
    'homeassistant': 'https://homeassistant.imanizadyar.com',
    'homepage': 'https://dashboard.imanizadyar.com',
    'jellyfin': 'https://jellyfin.imanizadyar.com',
    'affine': 'https://affine.imanizadyar.com',
    'nextcloud': 'https://nextcloud.imanizadyar.com',
    'prometheus': 'https://prometheus.imanizadyar.com',
    'metube': 'https://metube.imanizadyar.com',
    'snapotter': 'https://snapotter.imanizadyar.com',
    'dozzle': 'https://dockerlogs.imanizadyar.com'
  } %}
  {% for c in containers | sort(attribute='created', reverse=true) %}
  {% set service_url = service_urls.get(c.name) %}
  {% if c.state == 'running' %}
    {% set indicator = '🟢' %}
  {% elif c.state == 'restarting' %}
    {% set indicator = '🟠' %}
  {% elif c.state == 'paused' %}
    {% set indicator = '🟡' %}
  {% else %}
    {% set indicator = '🔴' %}
  {% endif %}
  | {% if service_url %}[{{ c.name }}]({{ service_url }}){% else %}`{{ c.name }}`{% endif %} | [{{ indicator }} {{ c.state }}](https://dockerlogs.imanizadyar.com/show?name={{ c.name }}) | {{ c.created_date }} | `{{ c.image }}` |
  {% endfor %}
grid_options:
  columns: 24
  rows: auto

```
{% endcode %}

<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

Now this is totally acceptable and will definitely work. Clicking the names leads you to the services and the status leads you to the logs. From a technical point of view, the docker containers have successfully been moved to the home assistant page.

I want to make this look a bit more visually appealing so I will make use of HomeAssistant Community Store (HACS). I don't remember how I installed it but the guides out there are pretty easy. Once installed, using the button-card @ RomRider and auto-entities @ thomasloven repos, you can take your cards to the next level with the following Auto Entities card configuration. ENSURE you are on the code editor in the bottom left of the configuration.

{% code title="Auto Entities card configuration (code editor)" expandable="true" %}
```yaml
type: custom:auto-entities
card:
  type: grid
  columns: 8
  square: false
card_param: cards
show_empty: false
filter:
  template: |
    [
      {% set containers = state_attr('sensor.docker_container_inventory', 'containers') or [] %}
      {% set init_names = [
        'livesync-init',
        'affine-migration'
      ] %}
      
      {% set running = containers
        | selectattr('state', 'equalto', 'running')
        | rejectattr('name', 'in', init_names)
        | sort(attribute='created', reverse=true)
        | list %}
      
      {% set stopped = containers
        | rejectattr('state', 'equalto', 'running')
        | rejectattr('name', 'in', init_names)
        | sort(attribute='created', reverse=true)
        | list %}
      
      {% set initialisation = containers
        | selectattr('name', 'in', init_names)
        | sort(attribute='created', reverse=true)
        | list %}
      {% set icons = {
        'homeassistant': 'mdi:home-assistant',
        'homepage': 'mdi:view-dashboard',
        'jellyfin': 'mdi:play-circle',
        'metube': 'mdi:download',
        'cloudflared': 'mdi:cloud',
        'snapotter': 'mdi:camera',
        'snapotter-postgres': 'mdi:database',
        'snapotter-redis': 'mdi:database',
        'nextcloud': 'mdi:cloud-outline',
        'nextcloud-db': 'mdi:database',
        'nextcloud-redis': 'mdi:database',
        'affine': 'mdi:note-edit',
        'affine-postgres': 'mdi:database',
        'affine-redis': 'mdi:database',
        'affine-migration': 'mdi:database-sync',
        'prometheus': 'mdi:chart-line',
        'dozzle': 'mdi:text-box-search',
        'dozzle-socket-proxy': 'mdi:shield-lock',
        'docker-socket-proxy': 'mdi:shield-lock',
        'livesync-couchdb': 'mdi:database',
        'livesync-tailscale': 'mdi:vpn',
        'livesync-init': 'mdi:cog',
        'the-union-bot': 'mdi:robot',
        'caddy': 'mdi:server-security'
      } %}
      {% set services = {
        'homeassistant': 'https://homeassistant.imanizadyar.com',
        'homepage': 'https://dashboard.imanizadyar.com',
        'jellyfin': 'https://jellyfin.imanizadyar.com',
        'affine': 'https://affine.imanizadyar.com',
        'nextcloud': 'https://nextcloud.imanizadyar.com',
        'prometheus': 'https://prometheus.imanizadyar.com',
        'metube': 'https://metube.imanizadyar.com',
        'snapotter': 'https://snapotter.imanizadyar.com',
        'dozzle': 'https://dockerlogs.imanizadyar.com'
      } %}
      {% for c in running + stopped + initialisation %}
        {% set status = c.state | default('unknown') %}
        {% set image = c.image.split('/')[-1] %}
        {% set colour =
          'rgba(120, 120, 120, 0.18)' if c.name in init_names
          else 'rgba(46, 125, 50, 0.18)' if status == 'running'
          else 'rgba(198, 40, 40, 0.18)' if status == 'exited'
          else 'rgba(120, 120, 120, 0.18)'
        %}
        {{
          {
            'type': 'custom:button-card',
            'entity': 'sensor.docker_container_inventory',
            'name': c.name,
            'icon': icons.get(c.name, 'mdi:docker'),
            'show_state': false,
            'show_label': true,
            'label': status | upper ~ ' · ' ~ image ~ ' · Created ' ~ c.created_date,
            'tap_action': {
              'action': 'url',
              'url_path': services.get(c.name, 'https://dockerlogs.imanizadyar.com/show?name=' ~ c.name)
            },
            'hold_action': {
              'action': 'url',
              'url_path': 'https://dockerlogs.imanizadyar.com/show?name=' ~ c.name
            },

            'styles': {
              'card': [
                {'background-color': colour},
                {'padding': '14px'},
                {'height': '140px'},
                {'border-radius': '12px'}
              ],
              'name': [
                {'font-size': '16px'},
                {'font-weight': '600'},
                {'white-space': 'normal'},
                {'overflow-wrap': 'anywhere'}
              ],
              'label': [
                {'font-size': '13px'},
                {'white-space': 'normal'},
                {'opacity': '0.85'}
              ],

            }
          }
        }},
      {% endfor %}
    ]
grid_options:
  columns: 48
  rows: auto

```
{% endcode %}

One thing you may note here is that I have manually assigned icons and services. This could be passed through the docker compose however since I do not constantly need services and icons updated on the home page, I will not go through the hassle. The columns at the top have been customised for my home page as you can see below

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

Clicking on a container will take you to its service (if it has one) otherwise holding it will take you to its logs.

