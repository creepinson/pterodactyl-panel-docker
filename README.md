# pterodactyl-panel-docker
[![GitHub release](https://img.shields.io/github/release/creepinson/pterodactyl-panel-docker.svg?style=for-the-badge)](https://github.com/creepinson/pterodactyl-panel-docker/releases)
> Docker Compose configuration for the [Pterodactyl Panel](https://github.com/Pterodactyl/Panel).

Pterodactyl is an open-source control panel used for hosting numerous game-related
services, such as Minecraft and Teamspeak servers. Setting it up, however, takes
quite a few steps according to the [documentation](https://docs.pterodactyl.io/docs).

This Docker Compose configuration is aimed at removing a majority of the steps so
that the installation process is *almost* as simple as cloning this build script and
running `docker-compose up`.

There are other Docker Compose setups available, including the [one](https://github.com/parkervcp/pterodactyl-panel-Dockerfile)
created by one of the project's primary developers, parkervcp. This differs from
those setups in various ways for a couple of different reasons:

1. This setup splits the HTTP and PHP services into two different containers. Doing
so allows for separation of concerns (a central Docker principle) and easier management
of containers and Dockerfiles.
2. This setup uses Nginx instead of Caddy. Various [benchmarks](https://community.centminmod.com/threads/caddy-http-2-server-benchmarks.5170/#post-34367)
[indicate](https://hackernoon.com/caddy-a-modern-web-server-vs-nginx-e9e4abc443e)
that Nginx performs better than Caddy in handling requests. In addition to this,
Nginx, while having a more complicated configuration file, doesn't enforce HTTPS,
a feature of Caddy which only complicates the setup of the HTTP Docker container.

## Usage
The instructions are fairly self explanatory: clone the repository and launch the
project. Instructions on how to launch the project are listed below. All commands
should be run from the repository directory.

In your favorite text editor, open docker-compose.yml change the port that you want
the panel binded to. See below for more details. Once that's done, build and start
the project.
```
docker-compose -p <project_name> pull
docker-compose -p <project_name> build
docker-compose -p <project_name> up
```
The panel is now online. If this is your first time booting the panel, you'll also
need to run the install script, which will run all of the necessary configuration
scripts and database migrations for you.
```
docker exec -it <project name>_php_1 install
# Restart the panel to load changes. The install script will remind you to do so.
docker restart <project name>_php_1
```
The panel should now be ready, and you can connect to it via localhost at the port
you specified in the Docker Compose file.

## API
To set the port for the HTTP server, adjust the setting under services, http, and
ports:
```
<host port number>:80
```
Two scripts are provided in order to aid with installation and upgrades.
```
# Use these scripts with the following command:
docker exec -it <project name>_php_1 <script name>
- install: Runs all the installation scripts, including migrations and configuration.
- upgrade: Runs all the steps needed to upgrade to the latest version of the panel.
```
Note that depending on how old your version of the panel is, the upgrade command
may not work. Be sure to check the changelog for each release and make sure your
version is not listed as not working.

### Upgrading
**Always remember to back up your data before upgrades! While I do my best to keep
this Docker Compose configuration stable, there is always the chance that something
can go horribly wrong! See the next section on how to run backups.**

Upgrading the project is slightly more complex. See the second caveat for an explanation
as to why.

To upgrade, take down the project and wipe out the panel volume as it interferes
with the upgrade process.
```
docker-compose -p <project name> down
docker volume rm <project name>_panel
```
Rebuild and restart the project.
```
docker-compose -p <project name> pull
docker-compose -p <project name> build
docker-compose -p <project name> up
```
Finally, run the upgrade script.
```
docker exec -it <project name>_php_1 upgrade
# Restart the panel to load changes. The upgrade script will remind you to do so.
docker restart <project name>_php_1
```

### Backups
The volumes that you will need to backup are `db` (database), `env` (configuration),
and `storage` (stored eggs and other files). These commands only work when the project
is up (a caveat of Docker).
```
# Use mysqldump to back up the database.
docker exec -it <project name>_db_1 mysqldump -u pterodactyl -ppterodactyl pterodactyl > database.sql
# You can copy the .env file from the mounted env volume.
docker cp <project name>_php_1:/var/www/html/env/.env .env
# You can copy the mounted storage volume.
docker cp <project name>_php_1:/var/www/html/pterodactyl/storage storage
```

## Caveats
- Due to the need for a .env file early on in the build process, it is impossible
to automatically run an install script, which would depend on checking for the existence
of a .env file. You will need to manually run the install and upgrade scripts as
necessary. Their use has been covered in the previous section.
- The panel volume is required to share data between the HTTP and PHP containers,
but the existence of this volume interferes with upgrade files due to how Docker
works. These containers are separated for reasons already outlined, but this does
mean that this volume must manually be removed on each upgrade.
- A symbolic link is used to maintain the state of the .env file rather than Docker
configs because of issues with **Docker itself**. Configs exist, but because they
are read-only filesystems, they cannot be modified in the container, making their
maintenance much more complicated. They would also require the use of Docker Swarms
and Docker Services.
- This setup is for Docker Compose, **not** for Docker Swarm or Docker Service. These
two Docker services take much more convoluted configuration, including the setup
of a swarm and the use of a local image registry, which would only complicate this
setup.
