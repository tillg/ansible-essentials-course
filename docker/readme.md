# Docker setup for *Ansible Essential* Udemy Course

## Getting started

In order to start the local docker network follow these steps:

```bash
git clone git@github.com:tillg/ansible-essentials-course.git
cd ansible-essentials-course/docker
docker-compose up --build &
```

Now your local network of nodes is running:
![Lab Topology](../lab-topology.png)

To connect to the controller via a terminal:

```bash
docker exec -it controller bash
```
### Accessing nodes

The different nodes of the docker-setup network run _inside_ the docker network. To get access to them, I exposed ports like so:

* lb expoises it's port 80 to 1001
* web1 resp. web2 expose their port 80 to 1011 resp. 1012

So to see what web2 serves, point your browser to [http://localhost:1012/](http://localhost:1012/)

## Problems & Solutions

### Containers stop 

When launching the docker network the containers start and stop immediatly. 

**Solution** In the start sequence of the container we simply add a `sleep infinity` statment - that keeps hiom alive 😀

**Solution (bad)** It turns out, that to keep them running we just have to add a command that keeps them _busy_: `command: tail -F anything` Found the solution [here](https://stackoverflow.com/questions/38546755/docker-compose-keep-container-running/45450456). This solution is very hacky and produces cluttered output, therefore I replaced it.

### SSH Connection

The controller docker at first could not ssh into the other dockers. And this is needed, even for the Ansible ping module (as the ping module creates a SSH connection - instead of actually pinging...).

**Solution** I replaced the CentOS docker images with a Ubuntu image, as I know more about Ubuntu. Then I created Dockerfiles for the controller and for the nodes that make sure that the SSH server and all the needed packages are installed. I also created start scripts that start the SSH server with the parameters needed (i.e. root user can login via ssh).

Careful, I changed the password of root users to `root` to make things easier. But it's not safe! 🤞

Helpful commands:

* See what services are running: `service --status-all`
* Check if ssh deamon is running: `service ssh status`
* Start ssh deamon: `service ssh start`
* What Linux version am I using?: `lsb_release -a`

### Accessing containers via IP 

I want to access my containers via individual IP addresses from my host, my host being a Mac. 

**Solution** After some trying and reading, I found out that I simply cannot:

**Per-container IP addressing is not possible**

The docker (Linux) bridge network is not reachable from the macOS host.

I found this in the Docker documentation [Networking features in Docker Desktop for Mac](https://docs.docker.com/docker-for-mac/networking/).

So the best work around is exposing certain ports that allow me to work with specific services on my containers.