# How It All Works

## Understanding TRMM

Anything you configure: scripts, tasks, patching etc is queued and scheduled on the server to do something. 
Everything that is queued, happens immediately when agents are online.
The agent gets a nats command, server tells it to do xyz and it does it.

When agents are not connected to the server nothing happens. The windows task scheduler says do x at some time, what it's asked to do is get x command from the server. If server is offline, nothing happens.
If an agent comes online, every x interval (windows update, pending tasks etc) check and see is there something for me to do that I missed while I was offline. When that time occurs (eg agent sees if it needs to update itself at 35mins past every hr [Update Agents](update_agents.md) ) it'll get requested on the online agent.

That's the simplified general rule for everything TRMM.

[![Network Design](images/TacticalRMM-Network.png)](images/TacticalRMM-Network.png)

Still need graphics for

    1. Agent installer steps

    2. Agent checks/tasks and how they work on the workstation/interact with server

## Server

Has a postgres database located here:

[Django Admin](functions/django_admin.md)

!!!description
    A web interface for the postgres database

All Tactical RMM dependencies are listed [here](https://github.com/amidaware/tacticalrmm/blob/develop/api/tacticalrmm/requirements.txt)

A complete list of all packages used by Tactical RMM are listed [here](https://raw.githubusercontent.com/amidaware/tacticalrmm/develop/web/package-lock.json)

### Outbound Firewall Rules

If you have strict firewall rules these are the only outbound rules from the server needed for all functionality:

1. Outbound traffic to all agent IP scopes for reflect traffic from agents

#### Server without Code Signing key

No additional rules needed

#### Server with Code Signing key

No additional rules needed

### System Services

This lists the system services used by the server.

#### nginx web server

Nginx is the web server for the `rmm`, `api`, and `mesh` domains. All sites redirect port 80 (HTTP) to port 443 (HTTPS).

!!! warning

    nginx does not serve the NATS service on port 4222.

???+ abstract "nginx configuration (a.k.a. sites available)"

    - [nginx configuration docs](https://docs.nginx.com/nginx/admin-guide/basic-functionality/managing-configuration-files/)

    === ":material-web: `rmm.example.com`"

        This serves the frontend website that you interact with.

        - Config: `/etc/nginx/sites-enabled/frontend.conf`
        - root: `/var/www/rmm/dist`
        - Access log: `/var/log/nginx/frontend-access.log`
        - Error log: `/var/log/nginx/frontend-error.log`
        - TLS certificate: `/etc/letsencrypt/live/example.com/fullchain.pem`

    === ":material-web: `api.example.com`"

        This serves the TRMM API for the frontend and agents. 

        - Config: `/etc/nginx/sites-enabled/rmm.conf`
        - roots:
            - `/rmm/api/tacticalrmm/static/`
            - `/rmm/api/tacticalrmm/tacticalrmm/private/`
        - Upstreams:
            - `unix://rmm/api/tacticalrmm/tacticalrmm.sock`
            - `unix://rmm/daphne.sock`
        - Access log: `/rmm/api/tacticalrmm/tacticalrmm/private/log/access.log`
        - Error log: `/rmm/api/tacticalrmm/tacticalrmm/private/log/error.log`
        - TLS certificate: `/etc/letsencrypt/live/example.com/fullchain.pem`

    === ":material-web: `mesh.example.com`"

        This serves MeshCentral for remote access.

        - Config: `/etc/nginx/sites-enabled/meshcentral.conf`
        - Upstream: `http://127.0.0.1:4430/`
        - Access log: `/var/log/nginx/access.log` (uses default)
        - Error log: `/var/log/nginx/error.log` (uses default)
        - TLS certificate: `/etc/letsencrypt/live/example.com/fullchain.pem`

    === ":material-web: default"

        This is the default site installed with nginx. This listens on port 80 only.

        - Config: `/etc/nginx/sites-enabled/default`
        - root: `/var/www/rmm/dist`
        - Access log: `/var/log/nginx/access.log` (uses default)
        - Error log: `/var/log/nginx/error.log` (uses default)

???+ note "systemd config"

    === ":material-console-line: status commands"

        - Status: `systemctl status --full nginx.service`
        - Stop: `systemctl stop nginx.service`
        - Start: `systemctl start nginx.service`
        - Restart: `systemctl restart nginx.service`
        - Restart: `systemctl reload nginx.service` reloads the config without restarting
        - Test config: `nginx -t`
        - Listening process: `ss -tulnp | grep nginx`

    === ":material-ubuntu: standard"

        - Service: `nginx.service`
        - Address: `0.0.0.0`
        - Port: 443
        - Exec: `/usr/sbin/nginx -g 'daemon on; master_process on;'`
        - Version: 1.18.0

    === ":material-docker: docker"

        - From the docker host view container status - `docker ps --filter "name=trmm-nginx"`
        - View logs: `docker-compose logs tactical-nginx`
        - "tail" logs: `docker-compose logs tactical-nginx | tail`
        - Shell access: `docker exec -it trmm-nginx /bin/bash`


#### Tactical RMM (Django uWSGI) service

Built on the Django framework, the Tactical RMM service is the heart of the system by serving the API for the frontend and agents.

???+ note "uWSGI config"

    - [uWSGI docs](https://uwsgi-docs.readthedocs.io/en/latest/index.html)

    === ":material-console-line: status commands"

        - Status: `systemctl status --full rmm.service`
        - Stop: `systemctl stop rmm.service`
        - Start: `systemctl start rmm.service`
        - Restart: `systemctl restart rmm.service`
        - journalctl:
            - "tail" the logs: `journalctl --identifier uwsgi --follow`
            - View the logs: `journalctl --identifier uwsgi --since "30 minutes ago" | less`
            - Debug logs for 5xx errors will be located in `/rmm/api/tacticalrmm/tacticalrmm/private/logs`

    === ":material-ubuntu: standard"
    
        - Service: `rmm.service`
        - Socket: `/rmm/api/tacticalrmm/tacticalrmm.sock`
        - uWSGI config: `/rmm/api/tacticalrmm/app.ini`
        - Log: None
        - Journal identifier: `uwsgi`
        - Version: 2.0.18
    
    === ":material-docker: docker"

        - From the docker host view container status - `docker ps --filter "name=trmm-backend"`
        - View logs: `docker-compose logs tactical-backend`
        - "tail" logs: `docker-compose logs tactical-backend | tail`
        - Shell access: `docker exec -it trmm-backend /bin/bash`

#### Daphne: Django channels daemon

[Daphne](https://github.com/django/daphne) is the official ASGI HTTP/WebSocket server maintained by the [Channels project](https://channels.readthedocs.io/en/stable/index.html).

???+ note "Daphne config"

    - Django [Channels configuration docs](https://channels.readthedocs.io/en/stable/topics/channel_layers.html)

    === ":material-console-line: status commands"

        - Status: `systemctl status --full daphne.service`
        - Stop: `systemctl stop daphne.service`
        - Start: `systemctl start daphne.service`
        - Restart: `systemctl restart daphne.service`
        - journalctl (this provides only system start/stop logs, not the actual logs):
            - "tail" the logs: `journalctl --identifier daphne --follow`
            - View the logs: `journalctl --identifier daphne --since "30 minutes ago" | less`

    === ":material-ubuntu: standard"

        - Service: `daphne.service`
        - Socket: `/rmm/daphne.sock`
        - Exec: `/rmm/api/env/bin/daphne -u /rmm/daphne.sock tacticalrmm.asgi:application`
        - Config: `/rmm/api/tacticalrmm/tacticalrmm/local_settings.py`
        - Log: `/rmm/api/tacticalrmm/tacticalrmm/private/log/debug.log`

    === ":material-docker: docker"

        - From the docker host view container status - `docker ps --filter "name=trmm-websockets"`
        - View logs: `docker-compose logs tactical-websockets`
        - "tail" logs: `docker-compose logs tactical-websockets | tail`
        - Shell access: `docker exec -it trmm-websockets /bin/bash`

#### NATS server service

[NATS](https://nats.io/) is a messaging bus for "live" communication between the agent and server. NATS provides the framework for the server to push commands to the agent and receive information back.

???+ note "NATS config"

    - [NATS server configuration docs](https://docs.nats.io/running-a-nats-service/configuration)

    === ":material-console-line: status commands"

        - Status: `systemctl status --full nats.service`
        - Stop: `systemctl stop nats.service`
        - Start: `systemctl start nats.service`
        - Restart: `systemctl restart nats.service`
        - Restart: `systemctl reload nats.service` reloads the config without restarting
        - journalctl:
            - "tail" the logs: `journalctl --identifier nats-server --follow`
            - View the logs: `journalctl --identifier nats-server --since "30 minutes ago" | less`
        - Listening process: `ss -tulnp | grep nats-server`

    === ":material-ubuntu: standard"
    
        - Service: `nats.service`
        - Address: `0.0.0.0`
        - Port: `4222`
        - Exec: `/usr/local/bin/nats-server --config /rmm/api/tacticalrmm/nats-rmm.conf`
        - Config: `/rmm/api/tacticalrmm/nats-rmm.conf`
            - TLS: `/etc/letsencrypt/live/example.com/fullchain.pem`
        - Log: None
        - Version: v2.3.3
    
    === ":material-docker: docker"
    
        - Get into bash in your docker with: `docker exec -it trmm-nats /bin/bash`
        - Log: `nats-api -log debug`
        - Shell access: `docker exec -it trmm-nats /bin/bash`

#### NATS API service

The NATS API service is a very light golang wrapper to replace traditional http requests sent to django. The agent sends the data to nats-api which is always listening for agent requests (on Port 4222). It then saves the data to postgres directly.

???+ note "NATS API config"

    === ":material-console-line: status commands"

        - Status: `systemctl status --full nats-api.service`
        - Stop: `systemctl stop nats-api.service`
        - Start: `systemctl start nats-api.service`
        - Restart: `systemctl restart nats-api.service`
        - journalctl: This application does not appear to log anything.

    === ":material-ubuntu: standard"
    
         - Service: `nats-api.service`
         - Exec: `/usr/local/bin/nats-api --config /rmm/api/tacticalrmm/nats-api.conf`
         - Config: `/rmm/api/tacticalrmm/nats-api.conf`
             - TLS: `/etc/letsencrypt/live/example.com/fullchain.pem`
         - Log: None
    
    === ":material-docker: docker"
    
        - Get into bash in your docker with: `docker exec -it trmm-nats /bin/bash`
        - Log: `nats-api -log debug`

#### Celery service

[Celery](https://github.com/celery/celery) is a task queue focused on real-time processing and is responsible for scheduling tasks to be sent to agents.

Log located at `/var/log/celery`

???+ note "celery config"

    - [Celery docs](https://docs.celeryproject.org/en/stable/index.html)
    - [Celery configuration docs](https://docs.celeryproject.org/en/stable/userguide/configuration.html)

    === ":material-console-line: status commands"

        - Status: `systemctl status --full celery.service`
        - Stop: `systemctl stop celery.service`
        - Start: `systemctl start celery.service`
        - Restart: `systemctl restart celery.service`
        - journalctl: Celery executes `sh` causing the systemd identifier to be `sh`, thus mixing the `celery` and `celerybeat` logs together.
            - "tail" the logs: `journalctl --identifier sh --follow`
            - View the logs: `journalctl --identifier sh --since "30 minutes ago" | less`
        - Tail logs: `tail -F /var/log/celery/w*-*.log`

    === ":material-ubuntu: standard"
    
        - Service: `celery.service`
        - Exec: `/bin/sh -c '${CELERY_BIN} -A $CELERY_APP multi start $CELERYD_NODES --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} --loglevel="${CELERYD_LOG_LEVEL}" $CELERYD_OPTS'`
        - Config: `/etc/conf.d/celery.conf`
        - Log: `/var/log/celery/w*-*.log`
    
    === ":material-docker: docker"
    
        - From the docker host view container status - `docker ps --filter "name=trmm-celery"`
        - View logs: `docker-compose logs tactical-celery`
        - "tail" logs: `docker-compose logs tactical-celery | tail`
        - Shell access: `docker exec -it trmm-celery /bin/bash`

#### Celery Beat service

[celery beat](https://github.com/celery/django-celery-beat) is a scheduler; It kicks off tasks at regular intervals, that are then executed by available worker nodes in the cluster.

???+ note "Celery Beat config"

    - [Celery beat docs](https://docs.celeryproject.org/en/stable/userguide/periodic-tasks.html)

    === ":material-console-line: status commands"

        - Status: `systemctl status --full celerybeat.service`
        - Stop: `systemctl stop celerybeat.service`
        - Start: `systemctl start celerybeat.service`
        - Restart: `systemctl restart celerybeat.service`
        - journalctl: Celery executes `sh` causing the systemd identifier to be `sh`, thus mixing the `celery` and `celerybeat` logs together.
            - "tail" the logs: `journalctl --identifier sh --follow`
            - View the logs: `journalctl --identifier sh --since "30 minutes ago" | less`
        - Tail logs: `tail -F /var/log/celery/beat.log`

    === ":material-ubuntu: standard"
    
        - Service: `celerybeat.service`
        - Exec: `/bin/sh -c '${CELERY_BIN} -A ${CELERY_APP} beat --pidfile=${CELERYBEAT_PID_FILE} --logfile=${CELERYBEAT_LOG_FILE} --loglevel=${CELERYD_LOG_LEVEL}'`
        - Config: `/etc/redis/redis.conf`
        - Log: `/var/log/celery/beat.log`
    
    === ":material-docker: docker"
    
        - From the docker host view container status - `docker ps --filter "name=trmm-celerybeat"`
        - View logs: `docker-compose logs tactical-celerybeat`
        - "tail" logs: `docker-compose logs tactical-celerybeat | tail`
        - Shell access: `docker exec -it trmm-celerybeat /bin/bash`

#### redis service

[redis](https://github.com/redis/redis) is an in-memory data structure store, used as a database, cache, and message broker for django/celery.

Log located at `/var/log/redis`

???+ note "redis config"

    - [Redis docs](https://redis.io/)

    === ":material-console-line: status commands"

        - Status: `systemctl status --full redis-server.service`
        - Stop: `systemctl stop redis-server.service`
        - Start: `systemctl start redis-server.service`
        - Restart: `systemctl restart redis-server.service`
        - Tail logs: `tail -F /var/log/redis/redis-server.log`

    === ":material-ubuntu: standard"
    
        - Service: `redis-server.service`
        - Log: `/var/log/redis/redis-server.log`
    
    === ":material-docker: docker"
    
        - From the docker host view container status - `docker ps --filter "name=trmm-redis"`
        - View logs: `docker-compose logs tactical-redis`
        - "tail" logs: `docker-compose logs tactical-redis | tail`
        - Shell access: `docker exec -it trmm-redis /bin/bash`
        
#### MeshCentral

[MeshCentral](https://github.com/Ylianst/MeshCentral) is used for: "Take Control" (connecting to machine for remote access), and 2 screens of the "Remote Background" (Terminal, and File Browser).

???+ note "meshcentral"

    - [MeshCentral docs](https://info.meshcentral.com/downloads/MeshCentral2/MeshCentral2UserGuide.pdf)

    === ":material-console-line: status commands"

        - Status: `systemctl status --full meshcentral`
        - Stop: `systemctl stop meshcentral`
        - Start: `systemctl start meshcentral`
        - Restart: `systemctl restart meshcentral`

    === ":material-docker: docker"

        - From the docker host view container status - `docker ps --filter "name=trmm-meshcentral"`
        - View logs: `docker-compose logs tactical-meshcentral`
        - "tail" logs: `docker-compose logs tactical-meshcentral | tail`
        - Shell access: `docker exec -it trmm-meshcentral /bin/bash`   

    === ":material-remote-desktop: Debugging"

        - Open either "Take Control" or "Remote Background" to get mesh login token
        - Open https://mesh.example.com to open native mesh admin interface
        - Left-side "My Server" > Choose "Console" > type `agentstats`
        - To view detailed logging goto "Trace" > click Tracing button and choose categories
     

#### MeshCentral Agent

Get Mesh Agent Version info with this command. Should match server version.

```cmd
"C:\Program Files\Mesh Agent\MeshAgent.exe" -info"
```
Compare the hash with the tags in the repo at <https://github.com/Ylianst/MeshAgent/tags>

### Other Dependencies

[Django](https://www.djangoproject.com/) - Framework to integrate the server to interact with browser.

<details>
  <summary>Django dependencies</summary>

```text
future==0.18.2
loguru==0.5.3
msgpack==1.0.2
packaging==20.9
psycopg2-binary==2.9.1
pycparser==2.20
pycryptodome==3.10.1
pyotp==2.6.0
pyparsing==2.4.7
pytz==2021.1
```
</details>

[qrcode](https://pypi.org/project/qrcode/) - Creating QR codes for 2FA.

<details>
  <summary>qrcode dependencies</summary>

```text
requests==2.25.1
six==1.16.0
sqlparse==0.4.1
```
</details>

[twilio](https://www.twilio.com/) - Python SMS notification integration.

<details>
  <summary>twilio dependencies</summary>

```text
urllib3==1.26.5
uWSGI==2.0.19.1
validators==0.18.2
vine==5.0.0
websockets==9.1
zipp==3.4.1
```
</details>


## Windows Agent

Found in `%programfiles%\TacticalAgent`

When scripts/checks execute, they are:

1. transferred from the server via nats
2. saved to a randomly created file in `c:\windows\temp\trmm\`
3. executed
4. Return info is captured and returned to the server via nats
5. File in `c:\windows\temp\trmm\` are removed automatically after execution/timeout.

### Outbound Firewall Rules

If you have strict firewall rules these are the only outbound rules from the agent needed for all functionality:

1. All agents have to be able to connect outbound to TRMM server on the 3 domain names on ports: 443 (agent and mesh) and 4222 (nats for checks/tasks/data)

2. The agent uses `https://icanhazip.tacticalrmm.io/` to get public IP info. If this site is down for whatever reason, the agent will fallback to `https://icanhazip.com` and then `https://ifconfig.co/ip`

#### Unsigned Agents

Unsigned agents require access to: `https://github.com/amidaware/rmmagent/releases/*`

#### Signed Agents

Signed agents will require: `https://agents.tacticalrmm.com` for downloading/updating agents

### Agent Installation Process

* Adds Defender AV exclusions
* Copies temp files to `c:\windows\temp\tacticalxxx` folder.
* INNO setup installs app into `%ProgramData%\TacticalAgent\` folder

***

### Agent Update Process

Downloads latest `winagent-vx.x.x-x86/64.exe` to `%programfiles%`

Executes the file (INNO setup exe)

Files create `c:\Windows\temp\Tacticalxxxx\` folder for install (and log files)

***

### Agent Debugging

#### Mesh Agent Recovery

#### Tactical Agent Recovery

### Windows Update Management

Tactical RMM Agent sets:

```reg
HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU
AUOptions (REG_DWORD):
1: Keep my computer up to date is disabled in Automatic Updates.
```

Uses this Microsoft API to handle updates: [https://docs.microsoft.com/en-us/windows/win32/api/_wua/](https://docs.microsoft.com/en-us/windows/win32/api/_wua/)

Server Queries Agent every 8hrs to check for update status.

### Log files

You can find 3 sets of detailed logs at `/rmm/api/tacticalrmm/tacticalrmm/private/log`

* `error.log` nginx log for all errors on all TRMM URL's: rmm, api and mesh

* `access.log` nginx log for access auditing on all URL's: rmm, api and mesh (_this is a large file, and should be cleaned periodically_)

* `django_debug.log` created by django webapp
