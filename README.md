# vaadin-shepherd2

Next attempt on Vaadin-Shepherd, built with off-the-shelf tools. Serves as a replacement
for Heroku, to publish your own stuff.

See the [previous Vaadin Shepherd](https://gitlab.vaadin.com/mavi/vaadin-shepherd).

This is mostly a documentation on how to get things running quickly in cloud VM.

Uses components:

* A VM with Ubuntu
* microk8s
* TeamCity

Shepherd expects the following from your project:

1. It must have `Dockerfile` at the root of its git repo.
2. The Docker image can be built via the `docker build --no-cache -t test/xyz:latest .` command
3. You need to write kubernetes yaml resource config file, which uses the image and defines all
   resources, services, ingresses etc.

Generally you place an appropriate `Dockerfile` to the root of your repository. See the following projects for an example:

1. Gradle+Embedded Jetty: [vaadin-embedded-jetty-gradle](https://github.com/mvysny/vaadin-embedded-jetty-gradle), [vaadin14-embedded-jetty-gradle](https://github.com/mvysny/vaadin14-embedded-jetty-gradle)
2. Maven+Embedded Jetty: [vaadin-embedded-jetty](https://github.com/mvysny/vaadin-embedded-jetty)
3. Gradle+WAR+Tomcat: [karibu10-helloworld-application](https://github.com/mvysny/karibu10-helloworld-application), [beverage-buddy-vok](https://github.com/mvysny/beverage-buddy-vok), [vok-security-demo](https://github.com/mvysny/vok-security-demo)

## Installing

Get a VM with 8 GB of RAM and Ubuntu x86-64, the newer the better. ssh into the machine as root & update.
Once you're in, we'll install and configure microk8s

```bash
$ apt install byobu snap
$ byobu
$ snap install microk8s --classic
$ microk8s enable dashboard dns registry ingress
```

Enable the dashboard, dns, registry and ingress modules according to [Playing with Kubernetes](https://mvysny.github.io/playing-with-kubernetes/)

Now, setup firewall:

```bash
ufw allow ssh
ufw allow in on cni0
ufw allow out on cni0
ufw default allow routed
ufw enable
ufw status
```

TODO setup TeamCity or Jenkins
