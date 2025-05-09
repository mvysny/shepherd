# Shepherd

Builds given git repos periodically and automatically deploys them to a Kubernetes cluster.
Serves as a homebrew "replacement" for Heroku, to publish your own pet projects. 
Built with off-the-shelf tools: Kubernetes and Jenkins.

See the [previous Vaadin Shepherd](https://gitlab.vaadin.com/mavi/vaadin-shepherd).

# Adding Your Project To Shepherd

> Tip: The [shepherd-cli](https://github.com/mvysny/shepherd-java-client) is a far easier way
> to add your projects. This way also works, but is more low-level, requires you to write kubernetes yaml config files
> and fiddle with Jenkins, and is more error-prone. shepherd-cli calls this project
> anyway, but its project config file is far simpler.

Shepherd expects the following from your project:

1. It must have `Dockerfile` at the root of its git repo.
2. The Docker image can be built via the `docker build -t test/xyz:latest .` command;
   The image can be run via `docker run --rm -ti -p8080:8080 test/xyz` command.
3. You can now register the project to Shepherd. **Continue to the "Adding a project" chapter** below.

Generally, all you need is to place an appropriate `Dockerfile` to the root of your project's git repository.
See the following projects for examples:

1. Gradle+Embedded Jetty packaged as zip: [vaadin-boot-example-gradle](https://github.com/mvysny/vaadin-boot-example-gradle),
   [vaadin14-boot-example-gradle](https://github.com/mvysny/vaadin14-boot-example-gradle),
   [karibu-helloworld-application](https://github.com/mvysny/karibu-helloworld-application),
   [beverage-buddy-vok](https://github.com/mvysny/beverage-buddy-vok),
   [vok-security-demo](https://github.com/mvysny/vok-security-demo)
2. Maven+Embedded Jetty packaged as zip: [vaadin-boot-example-maven](https://github.com/mvysny/vaadin-boot-example-maven)
3. Maven+Spring Boot packaged as executable jar: [Liukuri](https://github.com/vesanieminen/ElectricityCostDashboard),
   [my-hilla-app](https://github.com/mvysny/my-hilla-app), [vaadinplus](https://github.com/anezthes/vaadinplus), [TextField Formatter Zen](https://github.com/vaadin-component-factory/textfieldformatter-zen/)
   [Vaadin Hilla Spring PetClinic](https://github.com/jcgueriaud1/spring-petclinic-vaadin-flow)

## Maven+WAR

For Maven+war project, please use the following `Dockerfile`:

```dockerfile
# 1. Build the image with: docker build --no-cache -t test/xyz:latest .
# 2. Run the image with: docker run --rm -ti -p8080:8080 test/xyz

# The "Build" stage. Copies the entire project into the container, into the /app/ folder, and builds it.
FROM maven:3.9.1-eclipse-temurin-17 AS BUILD
COPY . /app/
WORKDIR /app/
RUN mvn -C clean test package -Pproduction
# At this point, we have the app WAR file in
# at /app/target/*.war
RUN mv /app/target/*.war /app/target/ROOT.war

# The "Run" stage. Start with a clean image, and copy over just the app itself, omitting gradle, npm and any intermediate build files.
FROM tomcat:10-jre17
COPY --from=BUILD /app/target/ROOT.war /usr/local/tomcat/webapps/
EXPOSE 8080
```

If your app fails to start, you can get the container logs by running:
```bash
$ docker exec -ti CONTAINER_ID /bin/bash
$ cat /usr/local/tomcat/logs/localhost.*
```

## Vaadin Addons

Vaadin addons are set up in a bit of an anti-pattern way:

* They package as jar, with the addon code only;
* The routes are packaged in the `src/test/` folder;
* They run via `mvn jetty:run`

The downside is that there's no support for production, and running via `mvn jetty:run`
requires Maven+Maven Repo+node_modules to be packaged in the docker image, increasing its size.

The solution is to:

* Replace Jetty with [Vaadin Boot](https://github.com/mvysny/vaadin-boot), adding Vaadin Boot as a test-scope dependency
* Creating a `Main.java` in `src/test/java/` which runs the app in Vaadin Boot.
* Creating Dockerfile which is able to build and run such app.

See [#16](https://github.com/mvysny/shepherd/issues/16) for more details; example project can be found at [parttio/breeze-theme](https://github.com/parttio/breeze-theme).

For addons that run via test-scoped Spring Boot, see the `Dockerfile` of the [parttio/parity-theme](https://github.com/parttio/parity-theme) example project.

# Shepherd Internals

This is a documentation on how to get things running quickly in cloud VM.

Shepherd needs/uses the following components:

* A VM with Ubuntu
* microk8s for keeping containers up-and-running, replaces nginx with ingress,
  contains logs, gathers metrics (CPU/Memory).
* Jenkins pulls git link periodically, then calls `shepherd-build` which builds new docker image, uploads to microk8s registry and restarts pods.
* no need for CD atm.

## Installing Shepherd

Get a VM with 8-16 GB of RAM and Ubuntu x86-64; use Ubuntu latest LTS.
ssh into the machine as root & update. Once you're in, we'll install and configure microk8s and jenkins.

First, install a bunch of useful utility stuff, then enter byobu:
```bash
$ apt update && apt -V dist-upgrade
$ apt install byobu snapd curl vim fish
$ byobu
$ sudo update-alternatives --config editor     # select vim.basic
```

Then, setup firewall, to shield ourselves during the followup installations steps. For example, Jenkins
by default listens on all interfaces - we don't want that:

```bash
$ ufw allow ssh
$ ufw enable
$ ufw status
```

### Jenkins

First, install Java since Jenkins depends on it:
```bash
$ apt install openjdk-11-jre
```
Then, [Install LTS Jenkins on Linux](https://www.jenkins.io/doc/book/installing/linux/) via apt.
That way, Jenkins integrates into SystemD and will start automatically when the machine is rebooted.

Check log to see that everything is okay: `journalctl -u jenkins`, `journalctl -u jenkins -f`.
Now, edit Jenkins config file via `systemctl edit jenkins` and add the following:
```
[Service]
Environment="JENKINS_LISTEN_ADDRESS=127.0.0.1"
Environment="JAVA_OPTS=-Djava.awt.headless=true -Xmx512m"
```

Restart Jenkins via `systemctl restart jenkins`.

ssh to the machine via `ssh -L localhost:8080:localhost:8080 -L localhost:10443:localhost:10443 root@xyz`,
then access Jenkins via [localhost:8080](http://localhost:8080), then
[Configure Jenkins](https://www.jenkins.io/doc/book/installing/linux/#unlocking-jenkins): 

* Disable all plugins except "Build Timeout", "Timestamper", "Git" and "Matrix Authorization Strategy".
* Create the `admin` user, with the password `admin`. This is okay since we'll need ssh port-forwarding to access Jenkins anyway.
* Set # of concurrent jobs: 2 or 3, depending on ubuntu memory (2 for 8GB, 3 for 16GB+)

### Docker

Then, install docker and add permissions to the Jenkins user to run it:

```bash
$ apt install docker.io docker-buildx
$ usermod -aG docker jenkins
$ reboot
```

### Microk8s

Install microk8s:
```bash
$ snap install microk8s --classic
$ microk8s disable ha-cluster --force
$ microk8s status
```

Disabling `ha-cluster` removes support for high availability & cluster but lowers the CPU usage significantly: [#1577](https://github.com/canonical/microk8s/issues/1577)

Setup firewall:

```bash
$ ufw allow in on cni0
$ ufw allow out on cni0
$ ufw default allow routed
$ ufw allow http
$ ufw allow https
$ ufw status
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere                  
Anywhere on vxlan.calico   ALLOW       Anywhere                  
Anywhere on cali+          ALLOW       Anywhere                  
Anywhere on cni0           ALLOW       Anywhere                  
80/tcp                     ALLOW       Anywhere                  
443                        ALLOW       Anywhere                  
22/tcp (v6)                ALLOW       Anywhere (v6)             
Anywhere (v6) on vxlan.calico ALLOW       Anywhere (v6)             
Anywhere (v6) on cali+     ALLOW       Anywhere (v6)             
Anywhere (v6) on cni0      ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)             
443 (v6)                   ALLOW       Anywhere (v6)             

Anywhere                   ALLOW OUT   Anywhere on vxlan.calico  
Anywhere                   ALLOW OUT   Anywhere on cali+         
Anywhere                   ALLOW OUT   Anywhere on cni0          
Anywhere (v6)              ALLOW OUT   Anywhere (v6) on vxlan.calico
Anywhere (v6)              ALLOW OUT   Anywhere (v6) on cali+    
```

Install more stuff to microk8s and setup user access:

```
$ microk8s enable dashboard
$ microk8s enable dns
$ microk8s enable registry
$ microk8s enable ingress:default-ssl-certificate=v-herd-eu-welcome-page/v-herd-eu-ingress-tls
$ microk8s enable cert-manager
$ usermod -aG microk8s jenkins
```

Add `alias mkctl="microk8s kubectl"` to `~/.config/fish/config.fish`

Verify that microk8s is running:
```
$ microk8s dashboard-proxy
```

(More commands & info at [Playing with Kubernetes](https://mvysny.github.io/playing-with-kubernetes/) ).

### Shepherd

To install Shepherd scripts, run:

```bash
$ cd /opt && sudo git clone https://github.com/mvysny/shepherd
```
Everything is now configured. To update Shepherd scripts, simply run

```bash
$ cd /opt/shepherd && sudo git pull --rebase
```

### Docker Build Cache

We need docker-buildx to enable docker build cache. To take advantage of build caches,
the project must contain this in their `Dockerfile`s:
```
RUN --mount=type=cache,target=/root/.gradle --mount=type=cache,target=/root/.vaadin ./gradlew clean build -Pvaadin.productionMode --no-daemon --info --stacktrace
```
Note the `--mount-type` arg.

We need separate build caches per project, otherwise we can't build projects in parallel since
they would overwrite their caches. See [#3](https://github.com/mvysny/shepherd/issues/3) for more details.

```bash
sudo mkdir -p /var/cache/shepherd/docker
sudo chgrp docker /var/cache/shepherd/docker
sudo chmod g+w /var/cache/shepherd/docker
```
Edit `/etc/docker/daemon.json` and enable containerd-snapshotter:
```json
{
  "features": {
    "containerd-snapshotter": true
  }
}
```
Restart docker daemon:
```bash
$ systemctl restart docker.service
```
Verify that the setting took effect:
```bash
$ docker info|grep driver-type
driver-type: io.containerd.snapshotter.v1
```

### Enabling HTTPS/SSL

Follow the Certbot/Let's Encrypt: https://microk8s.io/docs/addon-cert-manager tutorial.
The tutorial doesn't explain much, but it definitely works. Explanation here:
[Let's Encrypt HTTPS/SSL for Microk8s](https://mvysny.github.io/microk8s-lets-encrypt/):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: lets-encrypt
spec:
  acme:
    email: my-email
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
      - http01:
          ingress:
            class: public
```

We need to share one secret for `v-herd.eu` across multiple namespaces (multiple apps
mapping via ingress to `https://v-herd.eu/app1`. All solutions are at
[Cert Manager: syncing secrets across namespaces](https://cert-manager.io/docs/tutorials/syncing-secrets-across-namespaces/)

We'll solve this by reconfiguring the [Nginx default certificate](https://kubernetes.github.io/ingress-nginx/user-guide/tls/#default-ssl-certificate).
First we'll create a simple static webpage which makes CertManager
obtain the certificate from Let's Encrypt and store secret to
`v-herd-eu-welcome-page/v-herd-eu-ingress-tls`: [welcome-page.yaml](welcome-page.yaml):

```bash
$ mkctl apply -f welcome-page.yaml
```

After a while, https should start working; test it out [https://v-herd.eu](https://v-herd.eu).

We already registered the `--default-ssl-certificate=v-herd-eu-welcome-page/v-herd-eu-ingress-tls` option in the `nginx-controller` deployment,
when we enabled `ingress` above. You can verify that the configuration took effect, by
taking a look at the ` nginx-ingress-microk8s-controller` DaemonSet in microk8s Dashboard.

To configure the welcome page shown when browsing to `https://v-herd.eu`, go to the `v-herd-eu-welcome-page/static-site-vol` volume
folder. The folder should be at `/var/snap/microk8s/common/default-storage/v-herd-eu-welcome-page-static-site-pvc-*`,
see the [microk8s storage docs](https://microk8s.io/docs/addon-hostpath-storage) for details.
Example of the `index.html` can be found at [#12](https://github.com/mvysny/shepherd/issues/12).

### After Installation

If unchecked, docker build images will consume all disk space. Add the following cron weekly job to purge the images:

```bash
$ ln -s /opt/shepherd/shepherd-clearcache /etc/cron.weekly/
```

#### Exposing Jenkins via https

First, change Jenkins password to something more powerful. Then,
set this password in `/etc/shepherd/java/config.json` so that shepherd-cli
can still manage the setup. Then reconfigure jenkins context root to `/jenkins`
as described at [Jenkins behind reverse proxy](https://mvysny.github.io/jenkins-behind-reverse-proxy/).

We'll setup nginx to unwrap https and redirect traffic to Jenkins. First,
install nginx via `sudo apt install nginx-full`. Then, setup certificate retrieval
as described at [Let's Encrypt+Microk8s+nginx](https://mvysny.github.io/microk8s-lets-encrypt/),
chapter "nginx".

Edit `/etc/nginx/sites-available/default` and make it look like this:
```
server {
    listen 8443 ssl default_server;
    listen [::]:8443 ssl default_server;
    ssl_certificate /etc/nginx/secret/tls.crt;
    ssl_certificate_key /etc/nginx/secret/tls.key;

    server_name _;

    location /jenkins/ {
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host v-herd.eu;
        proxy_set_header X-Forwarded-Port 8443;
        proxy_pass http://localhost:8080;
        # proxy_cookie_domain localhost $host;  # not necessary?? Maybe Jenkins produces correct cookies thanks to X-Forwarded or other settings
    }
}
```

Then, `sudo systemctl reload nginx`. Jenkins is now accessible at `https://v-herd.eu:8443/jenkins`.

## Using Shepherd

Documents the most common steps after Shepherd is installed.

### Adding a project

First, decide on the project id, e.g. `vaadin-boot-example-gradle`. The project ID will go into k8s namespace;
Namespace must be a valid [DNS Name](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names),
which means that the project ID must:

* contain at most 54 characters (63 characters DNS name, minus 9 characters `shepherd-` prefix)
* contain only lowercase alphanumeric characters or '-'
* start with an alphanumeric character
* end with an alphanumeric character

Now call `shepherd-new vaadin-boot-example-gradle 256Mi` to create the project's k8s
resource config file yaml (named `/etc/shepherd/k8s/PROJECT_ID.yaml`).
See chapter below on tips on k8s yaml contents, for mem/cpu, env variables, database, Vaadin monitoring, persistent storage, ...

Now, create a Jenkins job:

* New Item, "vaadin-boot-example-gradle", Freestyle project, OK.
* Discard old builds, Max # of builds to keep=3
* Poll SCM with schedule `H/5 * * * *`
* Build Environment: Add timestamps to the Console Output
* Terminate a build if it's stuck; after 15 minutes. No timeout actions needed -> defaults to 'abort'
* Execute shell: `export BUILD_MEMORY=1500m && /opt/shepherd/shepherd-build vaadin-boot-example-gradle`
* Post-build: send e-mail
* Save, Build Now

The `shepherd-build` builder will copy the resource yaml, modify image hash, then `mkctl apply -f`.

Optionally, add the following env variables to the `shepherd-build`:

* `BUILD_MEMORY`: (optional) how much memory the build image will get. Defaults to `1024m`, but for Gradle use `1500m`
* `BUILD_ARGS`: (optional) for example add `export BUILD_ARGS='--build-arg offlinekey=foo'` to Jenkins.
  Then, add the following to your `Dockerfile`: `ARG offlinekey`; `ENV VAADIN_OFFLINE_KEY=$offlinekey` to pass in
  the Vaadin offline key to perform the production build with.
* `DOCKERFILE`: (optional) alternative name of the Dockerfile file

> Important: Make sure to use the "Server license key" and NOT the "Offline development license key":
> The offline dev key depends on the machine ID which may change easily when building in Docker.

#### k8s resource file contents tips

* Don't forget to use the `shepherd-PROJECT_ID` namespace.
* Since all projects have their own namespace, the k8s resource names can simply
  always be "deployment", "service", "pod", "ingress-global", "ingress-host".
* To modify mem/cpu resource limits, edit the yaml file Deployment part, the `spec.template.spec.containers[0].resources.limits` part

To expose a project on additional DNS domain, add:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-custom-dns-vaadin3-fake
  namespace: shepherd-PROJECT_ID    # use the right app namespace! 
  annotations:
    cert-manager.io/cluster-issuer: lets-encrypt
spec:
  tls:
    - hosts:
      - vaadin3.fake
      secretName: vaadin3-fake-ingress-tls
  rules:
    - host: "vaadin3.fake"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service
                port:
                  number: 8080
```

[Environment variables in Kubernetes](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/):

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: main
          image: <<IMAGE_AND_HASH>>
          env:
          - name: VAADIN_OFFLINE_KEY
            value: "[contents of offlineKey file here]"
```

##### Adding persistent PostgreSQL database

Please read the [Vaadin app with persistent PostgreSQL in Kubernetes](https://mvysny.github.io/kubernetes-vaadin-app-postgresql/)
for information on this setup. In short, add the following yaml to the app's kubernetes config yaml file:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: shepherd-TODO
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 512Mi } }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql-deployment
  namespace: shepherd-TODO
spec:
  selector:
    matchLabels:
      app: postgres-pod
  template:
    metadata:
      labels:
        app: postgres-pod
    spec:
      volumes:
        - name: postgres-vol
          persistentVolumeClaim:
            claimName: postgres-pvc
      containers:
        - name: postgresql
          image: postgres:15.2
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              value: mysecretpassword
          resources:
            requests:
              memory: "2Mi"
              cpu: 0
            limits:
              memory: "128Mi"
              cpu: "500m"
          volumeMounts:
            - name: postgres-vol
              mountPath: /var/lib/postgresql/data
---
apiVersion: v1
kind: Service
metadata:
  name: postgres  # this will also be the DNS name of the VM running this service.
  namespace: shepherd-TODO
spec:
  selector:
    app: postgres-pod
  ports:
    - port: 5432
```

1. Don't forget to change the `shepherd-TODO` namespace to the appropriate namespace of your app.
2. Configure your app to connect to the `jdbc:postgresql://postgres:5432/postgres` URL, with the `postgres` username and
   `mysecretpassword` password.

##### Spring Security

Spring Security introduces a servlet filter which uses HTTP 302 to redirect the user to the login page.
Unfortunately that redirect rule is not rewritten by ingress by default, causing the app
to redirect to `https://v-herd.eu/login` instead to `https://v-herd.eu/yourapp/login`. The fix is easy, just
add the following rewrite rules to the Ingress `metadata/annotations/` list of the yaml config file:

* `nginx.ingress.kubernetes.io/proxy-redirect-from`: `https://v-herd.eu/`
* `nginx.ingress.kubernetes.io/proxy-redirect-to`: `https://v-herd.eu\$1`

See [#18](https://github.com/mvysny/shepherd/issues/18) for more details.

##### More tips

TODO: Vaadin monitoring, ...

#### Manual changes to the project kubernetes yaml config file

The app Kubernetes config yaml file is located at `/etc/shepherd/k8s/PROJECT_ID.yaml`.
You can freely edit the file but the changes will be applied automatically only after the app is
built in Jenkins. To apply the changes faster, run the `./shepherd-apply PROJECT_ID` script manually from bash.

### Removing a project

* Remove CI pipeline by hand
* run `mkctl delete -f /etc/shepherd/k8s/PROJECT_ID.yaml`

## Shepherd Administration

ssh to the machine with proper port forwarding:

```bash
$ ssh -L localhost:8080:localhost:8080 -L localhost:10443:localhost:10443 root@xyz
$ byobu
$ microk8s dashboard-proxy
```

Browse:

* Jenkins: [localhost:8080](http://localhost:8080)
* Microk8s: [localhost:10443](http://localhost:10443)

## Shepherd API

Work in progress - will add more.

To list all projects, simply list the contents of the `/etc/shepherd/k8s/` folder. There will
be a bunch of yaml files corresponding to individual projects. The yaml naming is `PROJECT_ID.yaml`, so
you can obtain the project ID from the yaml file name.

In other words:

* A project with ID `foo` will have a file named `/etc/shepherd/k8s/foo.yaml`
* It will be exposed at `https://v-herd.eu/foo`
* TODO How to obtain run logs from all services from Kubernetes
* TODO how to obtain build logs from Jenkins

# Misc

## Troubleshooting

If you browse to the app, and you'll get nginx 404:

* Go to [Kubernetes Dashboard: Ingress](https://127.0.0.1:10443/#/ingress?namespace=_all) and make sure `Endpoints` shows `127.0.0.1`
  * If it doesn't, remove `kubernetes.io/ingress.class: nginx` from your ingress yaml, remove the ingress rule via `mkctl delete -f` and add it back.
  * Then, wait a bit - the ingress rule starts with empty endpoint but should set itself to `127.0.0.1` in a couple of seconds.

If you browse to the app, it does nothing, and then you'll get nginx 504:

* Try disabling `ufw` whether it helps.
  * If yes, try to enable the firewall back `ufw enable` and browse the app again - this usually helps.
  * If that doesn't help, try `ufw disable && ufw reset`, then re-add all rules back, then `ufw enable`.

If microk8s uses lots of CPU

* Disable `ha-cluster`: [#8](https://github.com/mvysny/shepherd/issues/8)

More troubleshooting tips:

* [microk8s & let's encrypt](https://mvysny.github.io/microk8s-lets-encrypt/)

If you get `No ED25519 host key is known for xyz.com and you have requested strict checking.` in Jenkins:

* Obtain the ED25519 host key. You can use the `ssh-keygen -l -F xyz.com` command if the host key is already on your dev machine.
* ssh to `v-herd.eu`, then run `sudo su jenkins`, then `ssh xyz.com`. ssh will print the key fingerprint; if it matches, press y.
  The ssh command may fail, but the key is now stored in `/var/lib/jenkins/.ssh/known_hosts`.
* Re-run Jenkins job.

## When Build Fails

* If you'll get a `java.text.ParseException: Invalid JWT serialization: Missing dot delimiter(s)`: the `VAADIN_OFFLINE_KEY` env variable may be
  empty. Make sure that you have `ARG offlinekey` in your `Dockerfile` and that you're passing the contents of the key correctly to `shepherd-build`
  using `BUILD_ARGS` as described above.
* `Unexpected exit value: 137` means that the build needs more memory. Increase the value of the `BUILD_MEMORY` env variable passed to `shepherd-build`.

## Configuration

Configuration: Every project has its k8s resource configuration file in `/etc/shepherd/k8s/`:

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
