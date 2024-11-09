# Raspberry Pi Docker Swarm instructions

## Preparing a Generic SD Card

See [Preparing SD Card](./docs/preparing_generic_sd_card.md) to setup a generic SD Card
for a swarm PI, unless someone else has prepared one for you.

## Customising SD Card for you Site

1. Insert the Generic SD card from the step above into a card reader
   connected to your computer.
2. Your computer should detect the card and you should be able to find a new
   drive in your file manager. On MacOS this will be called "system-boot"
3. Open the file `user-data` in a text editor and find the line
   `hostname: raspberrypi`. Change the word `raspberrypi` to a hostname
   likely to be unique on your local network. If you're setting up a site
   called `mysite`, we recommend `mysite1` as a hostname (and this will
   be used as an example in following instructions).
4. Open up the file `network-config` in a text editor. Find the section
   with the line `access-points:`. Underneath that, any number of WiFi
   network details can be added. Note that this file is a text file in
   the [YAML](https://en.wikipedia.org/wiki/YAML) format. This means that
   you need to indent sections inside `access-points` by two spaces
   (not a tab), and use quote marks `""` around your network name if it
   contains spaces.

## Raspberry Pi first boot

1. Insert a configured SD card into your Raspberry Pi, and connect ethernet,
   then power
2. Wait a few minutes, and then you should be able to run `ssh
   pi@mysite1.local` (replace the hostname with whatever hostname you set)
   with the default password `mbt`

## Setting up your swarm

```bash
$ ssh pi@mbt1.local ip -4 a show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.11.102/24 metric 100 brd 192.168.11.255 scope global dynamic eth0
       valid_lft 42906sec preferred_lft 42906sec
$ ssh pi@mbt1.local docker swarm init --advertise-addr 192.168.11.102
Swarm initialized: current node (fc3148voxuromdn6ab0p6zayb) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2y5on6clnwj9yxqlfqqdba68aai1ntcjdx5t9bn5mbznplhpt7-bqj26vi28yfdrq58lhiz3ky86 192.168.11.102:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

$ ssh pi@mbt2.local docker swarm join --token SWMTKN-1-2y5on6clnwj9yxqlfqqdba68aai1ntcjdx5t9bn5mbznplhpt7-bqj26vi28yfdrq58lhiz3ky86 192.168.11.102:2377
This node joined a swarm as a worker.
```

[Install Swarmpit](https://github.com/swarmpit/swarmpit#installation):

```bash
$ ssh -t pi@mbt1.local docker run -it --rm \
  --name swarmpit-installer \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  swarmpit/install:1.9
```

Configure Traefik for HTTP(S) proxying

```bash
docker network create --attachable --driver overlay proxy
# Edit traefik.yml and replace TOP_DOMAIN and TRAEFIK_ACME_EMAIL with something
# sensible
docker stack deploy --detach --compose-file traefik.yml traefik
```

---

> [!WARNING]
> Here be dragons!
> The below instructions haven't been polished yet and are just mattcen's notes

[Install and configure GlusterFS](https://thenewstack.io/tutorial-create-a-docker-swarm-with-persistent-storage-using-glusterfs/)

```bash
sudo apt install glusterfs-server
sudo systemctl enable --now glusterd
sudo gluster peer probe mbt2  # Run on mbt1
sudo gluster pool list  # Just a check
sudo mkdir -p /srv/gluster/vol1
sudo gluster volume create vol1 replica 2 \
  mbt1:/srv/gluster/vol1 mbt2:/srv/gluster/vol1 force  # On mbt1
sudo gluster volume start vol1  # On mbt1
sudo mkdir /vol1
echo 'localhost:/vol1 /vol1 glusterfs defaults,_netdev,backupvolfile-server=localhost 0 0' \
  | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
sudo chown -R root:docker /vol1
sudo chmod g+w /vol1
```

Set up Kiwix

```bash
pi@mbt1:~$ mkdir /vol1/kiwix
pi@mbt1:~$ wget -nv -P /vol1/kiwix https://download.kiwix.org/zim/wikipedia/wikipedia_en_100_2024-06.zim
2024-11-08 01:05:05 URL:https://md.mirrors.hacktegic.com/kiwix-md/zim/wikipedia/wikipedia_en_100_2024-06.zim [386004857/386004857] -> "/vol1/kiwix/wikipedia_en_100_2024-06.zim" [1]
docker stack deploy --detach --compose-file kiwix-wiki-100.yml kiwix-wiki-100
```
