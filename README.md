# Unofficial project
This repository provides Docker and Wine scripts to run the Eyeblink Windows application on Linux.
It is **not affiliated with, endorsed by, or maintained by Blinking Matters**.

This repository is built on top of [devcontainer](https://github.com/predkambrij/devcontainer).

## Commands
Start Docker container:
```bash
.assets/gen_env.sh
docker compose up -d --build
.assets/ssh.sh # by default password equal to the username
```

SSH session:
```bash
winecfg # then just click OK
wine reg add 'HKCU\Software\Wine\Explorer\Desktops' /v Default /d 3840x2160 /f # change for your resolution
wget https://www.blinkingmatters.com/files/download/eyeblink_setup.exe
wine eyeblink_setup.exe # Next, Next, Install, uncheck "Run eyeblink", Finish.

# Test it out (configure camera, reminder position, and other settings)
# Important: disable "Screen control", otherwise it won't find camera on 2nd loop (error: "0190:err:ole:CoUninitialize Mismatched CoUninitialize")
wine '.wine/drive_c/Program Files/Andrej Fogelton/eyeblink/eyeblink.exe'
```

Exit eyeblink, exit the SSH session. Add the following to `.assets/supervisord.conf`:
```
[program:eyeblink]
user=%(ENV_ARG_UNAME)s
directory=/home/%(ENV_ARG_UNAME)s
environment=HOME="/home/%(ENV_ARG_UNAME)s"
command = wine '.wine/drive_c/Program Files/Andrej Fogelton/eyeblink/eyeblink.exe'
autostart=true
autorestart=true
priority=1
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
```

Run and the program should start (with autorestart if it crashes).
```bash
docker compose up -d --build --force-recreate
```

To see eyeblink's logs:
```bash
docker compose logs -f
```

App logs:
```bash
tail -f ~/.wine/drive_c/users/$USER/AppData/Local/eyeblink/logs/$(ls -t ~/.wine/drive_c/users/$USER/AppData/Local/eyeblink/logs/ | head -n 1)
```

# Known limitations
- Browser links won't work, since the program is running in a Docker container. This repo is testing out what steps are required to make the program work. It's more convenient to run it outside the Docker container.
- The program is using lots of CPU (probably because GPU acceleration is not working correctly).
- if camera gets reconnected you need to restart te docker container

---
# DevContainer README.md
# Description
This repository contains scripts that makes it easy to start working on random projects without installing a bunch of dependencies on the host computer.

## generate .env

    .assets/gen_env.sh

## build

    docker compose build


## create network if it doesn't exist yet

    bash -c '. .env; docker network ls --format "{{.Name}}" | grep -Fxq "${NETWORK_NAME}" || docker network create "${NETWORK_NAME}"'

## start the container

    docker compose up -d

# SSH

By default password is equal to username. You can change it, or remove it and use ssh key.

## ssh to the container

    .assets/ssh.sh

## other ways to ssh

### using network_mode: host
This one might be the most convenient, especially if you need to run some services on localhost, but want to access them from the host machine. It might be a bit annoying to deal with port conflicts.

Uncomment `network_mode: host` and add the following to dockerfile_inline:

    RUN echo 'Port 2022' >> /etc/ssh/sshd_config

Add the following to your ssh config (example):

    Host devbox_test-devbox-1
        Hostname localhost
        Port 2022

Run `docker compose up -d --build` and then ssh:

    ssh devbox_test-devbox-1

### using hoster
[dvddarias/docker-hoster](https://github.com/dvddarias/docker-hoster) will insert container name in `/etc/hosts` so you can `ssh <containername>` which is handy, just run `docker ps -a` to get container name first.

### using dnsdock with dnsmasq
[aacebedo/dnsdock](https://github.com/aacebedo/dnsdock) provides DNS resolution.

Start the following

    docker run --restart=unless-stopped -d -v /var/run/docker.sock:/var/run/docker.sock --name dnsdock -p 127.0.1.53:53:53/udp aacebedo/dnsdock:v1.17.0-amd64 -v --domain=docker

Add the following to `/etc/dnsmasq.conf`

    server=/docker/127.0.1.53

Then you can add to ssh config (example):

    Host devbox_test-devbox-1
        Hostname devbox_devbox_test.docker

Then ssh:

    ssh devbox_test-devbox-1


## ssh to devcontainer on another machine
You can uncomment ports section and `~/.Xauthority` section in docker-compose.yml and do the following

    ssh -Y -p 2022 $USER@<hostmachine>

