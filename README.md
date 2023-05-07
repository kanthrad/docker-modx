MODX Docker Project
---

*This project is a copy of [bezumkin/modx-docker](https://github.com/bezumkin/modx-docker)*

Download project from Gitlab:

```
git clone https://github.com/kanthrad/docker-modx.git ./new-project-name
```

### Prepare Docker

If you have no Docker, please install it with
```
brew install docker --cask
brew install docker-compose
```
Then launch Docker desktop application from Applications.

Go to Docker directory:
```
cd ./new-project-name/docker
```

Prepare environment variables:
```
cp .env.dist .env
```

Specify unique project name in `COMPOSE_PROJECT_NAME` variable. Default: modx. This param also used as docker containers NAME prefix.

And run containers:
```
./start.sh
```

This script is launch a docker-compose.yml file to start: 
1. mariadb
1. njinx
1. php-fpm
1. node

First launch will take about 5-10 minutes while docker download images and build containers.

### Install MODX

If you have run this project for the first time, you need to install MODX with default settings. Run
```
./modx-install.sh
```

In this script the following steps happens:
1. download and extract MODX CMS archive to /modx folder
    - <span style="color:#d72849">ERROR</span>: "No such container: bash"
    - <span style="color:#aca5e7">TIPS</span>: replace $(docker ps --filter name=$NAME -q) with php-fpm docker container ID (see result of "docker ps" command in terminal)
1. launch script /setup/cli-install.php. This is  a default script in modx cms setup
    1. set install params
    1. include /setup/index.php file
        - <span style="color:#d72849">ERROR</span>: "Cannot start session when headers already sent..."
        - <span style="color:#aca5e7">TIPS</span>: comment session_start() function at the top of /setup/index.php file
1. remove /core/cache/ folder 
1. run ```gitify build``` command. This command used to read the data files (see /modx/_data/ folder), and write them to the MODX database
1. run ```gitify package:install``` command to install custom plugins (e.g., Ace, pdoTools)
    - <span style="color:#d72849">ERROR</span>: "Smarty: unable to create directory" (see /docker/log/njinx/error.log). Also because of that (i.e. wrong folder permision) http://127.0.0.1:8080 is 200 but there is no styles, scripts and http://127.0.0.1:8080/manager/ is answer with 500 errors
    - <span style="color:#aca5e7">TIPS</span>: set 777 permision to /core/cache folder ( e.g., ```sudo chmod -R 777 /modx/core/cache```)
1. <span style="color:#aca5e7">TIPS</span>: if there is still some error - try update Docker or check packages version compatibility

This will install MODX `2.8.4-pl`, default packages and create special `Assets` plugin. 

### How to develop

Open http://127.0.0.1:8080 - you will see the MODX website. 

Your frontend assets are in the `new-project-name/assets` directory, handled by Webpack in development mode. 
When you change files, frontend will rebuild assets and reload. 

If you want to change something in MODX, feel free to go to the `/manager` using login `admin` and password `adminadmin`.

When you finish your work, run 
```
./modx-extract.sh
```
to save your changes for Git.

Then you can stop your containers by
```
./stop.sh
```

### Production Build

If you want to upload compiled assets to production web-server, run
```
./modx-build.sh
```

This should compile frontend bundle and copy PHP sources with Gitify data files to the root `/dist` directory.

Now you are ready to upload the content of `/dist` directory to the root of MODX website on server.

## Windows notice

Although Docker works well on Windows, you can't run a bash script without installing WSL 2 or other complexities.

That is why you will need to run it directly inside PHP container. Open Docker Desktop, click on context menu of 
`php-fpm` container and use commands from scripts.

For example, here is all-in-one commands to install MODX:
```shell
export $(cat ./.env | sed 's/\r$//')

gitify modx:download 2.8.4-pl

php setup/cli-install.php --database_server=mariadb \
  --database=$MARIADB_DATABASE --database_user=$MARIADB_USERNAME --database_password=$MARIADB_PASSWORD \
  --table_prefix=modx_ --language=en --cmsadmin=admin --cmspassword=adminadmin --cmsadminemail=admin@localhost \
  --context_mgr_path=/modx/manager/ --context_mgr_url=/manager/ \
  --context_connectors_path=/modx/connectors/ --context_connectors_url=/connectors/ \
  --context_web_path=/modx/
  
rm -rf ./core/cache && gitify build

gitify package:install --all
```
