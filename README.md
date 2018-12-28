### How to enable Mac OS X native NFS (incl. Mojave)
And run app in Docker with native speed

Based on:
* https://medium.com/@sean.handley/how-to-set-up-docker-for-mac-with-native-nfs-145151458adc
* https://apple.stackexchange.com/a/337510

#### 0) Install Docker for Mac

Go to https://docs.docker.com/docker-for-mac/install/ and follow the installation instructions to get the latest version of Docker for Mac app.

#### 1) OS X Mojave users only: allow full disk access for your terminal app

1. Go to System Preferences > Security & Privacy > Privacy.
2. Click the "lock" icon to make changes.
3. Scroll down the list on the left-hand side and select "Full Disk Access"
4. Click the "+" icon on the right and select the Terminal app (or in my case iTerm).

#### 2) Create a setup file on you host machine

`setup_native_nfs_docker_osx.sh`:

```bash
#!/usr/bin/env bash

OS=`uname -s`

if [ $OS != "Darwin" ]; then
  echo "This script is OSX-only. Please do not run it on any other Unix."
  exit 1
fi

if [[ $EUID -eq 0 ]]; then
  echo "This script must NOT be run with sudo/root. Please re-run without sudo." 1>&2
  exit 1
fi

echo ""
echo " +-----------------------------+"
echo " | Setup native NFS for Docker |"
echo " +-----------------------------+"
echo ""

echo "WARNING: This script will shut down running containers."
echo ""
echo -n "Do you wish to proceed? [y]: "
read decision

if [ "$decision" != "y" ]; then
  echo "Exiting. No changes made."
  exit 1
fi

echo ""

if ! docker ps > /dev/null 2>&1 ; then
  echo "== Waiting for docker to start..."
fi

open -a Docker

while ! docker ps > /dev/null 2>&1 ; do sleep 2; done

echo "== Stopping running docker containers..."
docker-compose down > /dev/null 2>&1
docker volume prune -f > /dev/null

osascript -e 'quit app "Docker"'

echo "== Resetting folder permissions..."
U=`id -u`
G=`id -g`
sudo chown -R "$U":"$G" .

echo "== Setting up nfs..."
LINE="/Users -alldirs -mapall=$U:$G localhost"
FILE=/etc/exports
sudo cp /dev/null $FILE
grep -qF -- "$LINE" "$FILE" || sudo echo "$LINE" | sudo tee -a $FILE > /dev/null

LINE="nfs.server.mount.require_resv_port = 0"
FILE=/etc/nfs.conf
grep -qF -- "$LINE" "$FILE" || sudo echo "$LINE" | sudo tee -a $FILE > /dev/null

echo "== Restarting nfsd..."
sudo nfsd restart

echo "== Restarting docker..."
open -a Docker

while ! docker ps > /dev/null 2>&1 ; do sleep 2; done

echo ""
echo "SUCCESS! Now go run your containers 🐳"
```

Give it executable permissions and run without sudo:
```bash
$ chmod +x ./setup_native_nfs_docker_osx.sh
```

It will stop running Dockers containers and Docker service, enable NFS and then start Docker sevice back. You will be prompted for sudo password when enabling `nfsd` service.

#### 3) Update your `docker-compose.yml` according to the following example:

```yaml
version: '3'
services:
  api:
    volumes:
      - "nfsmount:/container/project/dir"

volumes:
  nfsmount:
    driver: local
    driver_opts:
      type: nfs
      o: addr=host.docker.internal,rw,nolock,hard,nointr,nfsvers=3
      device: ":/my/project/dir"
```

#### 4) Start your Docker project as usual:
```bash
$ docker-compose up
```

