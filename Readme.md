
# Deploy Portus and Docker Registry

Here are few steps to easily deploy [Portus](http://port.us.org) for your organisation.

Everything will be deployed as docker container in few commands.

Involved components:

* [Portus](http://port.us.org)
* [Docker Registry](http://dockr.ly/2vv9ZG4)
* [Nginx](https://nginx.org/en)
* [docker-gen](http://bit.ly/2vvcB6Z)
* [LetsEncrypt companion container for nginx-proxy](http://bit.ly/2vvoDgx)
* [mariaDB](https://mariadb.org/)

## Provision your cloud Instance

We will use _docker-machine_ to provision our cloud instance on DigitalOcean, but you can change the provider using a different driver (see [docker-machine driver](http://dockr.ly/2vv07wj) documentation). 

A. Let's boot your 5$/month instance:

```
$ docker-machine create --driver digitalocean portus
$ eval $(docker-machine env portus)
```

_Notes:_

- DigitalOcean driver env vars or aguments needed: [follow this...](http://dockr.ly/2vuLoBb)
- On 5$/month DO, you'll need a [swap file](##-Known-Issues:).

  ```
  $ docker-machine ssh portus
  ```

B. Then, create associated DNS with your instance public IP. It may take up to 72h.

Ex.: hub.example.com and registry.example.com

Modify those files accordingly:

- .env
- portus/portus.yml (especially machine_fqdn)

C. Create your TLS secure reverse-proxy:

```
$ docker network create webproxy
$ docker-compose -f webproxy.yml build
$ docker-compose -f webproxy.yml up -d
```

D. Start the Portus registry stack:

```
$ docker-compose -f portus-registry.yml build
$ docker-compose -f portus-registry.yml up -d db

# wait a bit you database start
# you can even inspect the logs:
$ docker-compose -f portus-registry.yml logs -f db
# stop streaming logs with ctrl+c

$ docker-compose -f portus-registry.yml up -d portus
# wait a bit portus db initialization ended (see logs)

$ docker-compose -f portus-registry.yml up -d crono
$ docker-compose -f portus-registry.yml up -d registry
```

E. check everything is up !

Open you browser:

- `https://hub.example.com`
- the default user:pass is: `portus:portus1234`

As first step, you'll have to [setup the registry link](http://bit.ly/2vv858s):

- Name: `registry`
- Hostname: `registry:5000`
- External Registry Name: `registry.example.com`

Then you're good to go and setup portus, just follow their [documentation](http://bit.ly/2vuNNMs)


Yeahaa, now let's push docker images !!!

## Known Issues:

- swapfile needed

  on a 5$ instance on DigitalOcean:

  The problem is that the server does not have enough memory to allocate for MariaDB process. There are a few solutions to this problem.

  1. Increase the physical RAM. Adding 1GB of additional RAM will solve the problem.

  2. Allocate SWAP space. Digital Ocean VPS instance is not configured to use swap space by default. By allocating 512MB of swap space, we were able to solve this problem. To add swap space to your server, please follow the following steps:

      As a root user, perform the following:

  ```
  $ dd if=/dev/zero of=/swap.dat bs=1024 count=524288
  $ chown root:root /swap.dat
  $ chmod 0600 /swap.dat
  $ mkswap /swap.dat
  $ swapon /swap.dat
  $ vi /etc/fstab
  ## Edit the /etc/fstab, and the following entry.
  /swap.dat none  swap  sw  0 0
  ```

  3. Reduce the size of MySQL buffer pool size

  Edit /etc/my.cnf, and add the following line

  ```
  [mysqld]
  innodb_buffer_pool_size=64M
  ```

  or starting _mysqld_ with the argument `--innodb-buffer-pool-size=64M` (already done in the portus-registry.yml)

