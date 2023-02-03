# vaadin-shepherd2

Next attempt on Vaadin-Shepherd, built with off-the-shelf tools. Serves as a replacement
for Heroku, to publish your own stuff.

See the [previous Vaadin Shepherd](https://gitlab.vaadin.com/mavi/vaadin-shepherd).

This is mostly a documentation on how to get things running quickly in cloud VM.

Uses components:

* A VM with Ubuntu
* microk8s for keeping containers up-and-running, replaces nginx with ingress,
  contains logs, gathers metrics (CPU/Memory).
* TeamCity pulls git link periodically, then calls `shepherd-build` which builds new docker image, uploads to microk8s registry and restarts pods.
* no need for CD atm.

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
$ apt install byobu snap openjdk-11-jre
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

TODO:
* setup TeamCity or Jenkins, but not in Docker so that they can access fs, run shepherd-build and run docker commands.
  * Try Jenkins first, [as a linux service](https://www.jenkins.io/doc/book/installing/linux/); TODO set memory to 2GB
  * max concurrent jobs: 2 or 3, depending on ubuntu memory (2 for 8GB, 3 for 16GB+)
  * max memory 1024m for Maven, 1500m for Gradle.
* Certbot/Let's Encrypt: https://microk8s.io/docs/addon-cert-manager
* Copy `scripts/` somewhere on the fs

## Adding a project

First, decide on the project id, e.g. `vaadin-boot-example-gradle`. The project ID will go into k8s namespace;
Namespace must be a valid [DNS Name](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names),
which means that the project ID must:

* contain at most 54 characters (63 characters DNS name, minus 9 characters `shepherd-` prefix)
* contain only lowercase alphanumeric characters or '-'
* start with an alphanumeric character
* end with an alphanumeric character

Now create the project's k8s resource config file yaml named `/etc/vaadin-shepherd/k8s/PROJECT_ID.yaml`.
Don't forget to use the `shepherd-PROJECT_ID` namespace. TODO use `shepherd-new` script to create a new yaml for you.
Since all project has its own namespace, the k8s resource names can simply always be "deployment", "service", "pod", "ingress-global", "ingress-host".

* That solves runtime config: mem, cpu, env variables, but also database, Vaadin monitoring, persistent storage, ...
* Builder will copy the resource yaml, modify image hash, then `mkctl apply -f`.

TODO create `shepherd-new` script which creates a new yaml for you, then
tell me next steps: to create the build pipeline in the CI which runs `shepherd-build`.

Add a job to teamcity. git-clone a project, then run `shepherd-build` on it. `shepherd-build` expects the following
env variables:

* `PROJECT_ID`: the project id, e.g. `vaadin-boot-example-gradle`
* `BUILD_MEMORY`: (optional) how much memory the build image will get. Defaults to `1024m`, but for Gradle use `1500m`

Doesn't accept buildargs, but the ENV variables can be defined directly in k8s resource yaml.

## Removing a project

* Remove CI pipeline by hand
* run `mkctl delete -f /etc/vaadin-shepherd/k8s/PROJECT_ID.yaml`

# Misc

## Configuration

Configuration: Every project has its own set of configuration files (perhaps committed to git?):

* microk8s: resource config yaml, defining:
   * routing/ingress/hostnames
   * container env (storage, env vars, resources: mem/cpu)
   * container name: `localhost:32000/shepherd/project_id`
* CI: git repo URL, build env (memory), docker buildArgs
   * CPUs per build: 2, max concurrent jobs: 3, max memory 1024m for Maven, 1500m for Gradle.
   * Builds docker images via `docker build --nocache -t localhost:32000/shepherd/project_id -m 1500m --cpu-period 100000 --cpu-quota 200000 --build-arg key=value .`
   * When it passes, it needs to construct new yaml for microk8s and push that, while deleting old rules.
      * This is where CD could help, but CD shared git repo sounds like trouble (read below).
      * Old rules won't be deleted by CI, only by hand: The only rule to be ever deleted is Ingress custom host rule, and that almost never happens.

## Tips for CI

[Tips on CI](https://mvysny.github.io/tips-on-ci/)
