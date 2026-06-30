# Homelab

Infrastructure experiments

## Current Stack
```
┌─────────────────────────────────────┐
│ Applications (Coolify, Gitea, etc.) │ 
├─────────────────────────────────────┤
│ Traefik (Reverse Proxy)             │ 
├─────────────────────────────────────┤
│ Ubuntu (OS)                         │ 
├─────────────────────────────────────┤
│ Proxmox (Virtualization)            │ 
└─────────────────────────────────────┘
```
## Hardware

| Component | Specification |
|-----------|---------------|
| **Processor** | Intel® Core™ i7-1060NG7 (10th Gen, 4C/8T) |
| **Base Speed** | 1.2 GHz (Burst: 3.8 GHz) |
| **Graphics** | Intel® Iris® Plus Graphics |
| **Memory** | 16GB LPDDR4 |
| **Storage** | 1TB NVMe SSD |

## Setup

### Proxmox VE 9.2
- Download ISO and creat bootable USB
- Installed on bare metal

### Ubuntu LXC Container
Created using [community script](https://community-scripts.org/scripts/ubuntu?from=scripts#install):
- 1 CPU core
- 512MB RAM
- 2GB disk

### Traefik

**Installation:**
```bash
# Download latest version (check for newer releases)
TRAEFIK_VERSION="3.7.5"
ARCH=$(dpkg --print-architecture 2>/dev/null || uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')

wget "https://github.com/traefik/traefik/releases/download/v${TRAEFIK_VERSION}/traefik_v${TRAEFIK_VERSION}_linux_${ARCH}.tar.gz"

# Install binary
sudo tar -xzf "traefik_v${TRAEFIK_VERSION}_linux_${ARCH}.tar.gz" -C /usr/local/bin

# Verify
traefik version
```

**Configuration:**

In Traefik can refer to two different things:

- 1) Install configuration/static configuration (startup): set up connections to **providers** and define the **entrypoints** Traefik will listen to (these elements don't change often).

- 2) Routing configuration/dynamic configuration: contains everything that defines how the requests are handled by your system. This configuration can change and is seamlessly hot-reloaded, without any request interruption or connection loss.

Traefik gets its routing configuration from providers: whether an orchestrator, a service registry, or a plain old configuration file.

1) **Install configuration**: There are three different, mutually exclusive (i.e. you can use only one at the same time), ways to define install configuration options in Traefik (in order):

    - 1) In a [configuration file](https://doc.traefik.io/traefik/getting-started/configuration-overview/#configuration-file)
    - 2) In the [command-line arguments](https://doc.traefik.io/traefik/getting-started/configuration-overview/#configuration-file)
    - 3) As [environment variables](https://doc.traefik.io/traefik/getting-started/configuration-overview/#configuration-file)


2) **Providing Dynamic (Routing) Configuration to Traefik**: defines how Traefik routes incoming requests to the correct services Depending on your environment and preferences, there are several ways to supply this routing configuration (checks [docs](https://doc.traefik.io/traefik/reference/routing-configuration/dynamic-configuration-methods/#providing-dynamic-routing-configuration-to-traefik)):
    - Use TOML or YAML files.
    - Docker and ECS Providers: Use container labels.
    - Kubernetes Providers: Use annotations.
    - KV Providers : Use key-value pairs.
    - Other Providers (Consul, Nomad, etc.) : Use tags.

  
 First try, config: 
 - 1. install config in a file: 
 - 2. routing config in a file

root user in ~ (/root)

Created a server /root/app/app.py:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()
        self.wfile.write(b"<h1>Hello from Traefik!</h1>")

server = HTTPServer(("127.0.0.1", 8080), Handler)
print("Running on http://127.0.0.1:8080")
server.serve_forever()
```
Run it in the background, send logs to file:
```bash
nohup python3 app/app.py > app/app.log 2>&1 &
curl localhost:8080
```
In `~/traefik/traefik.yaml`:
[entrypoints config](https://doc.traefik.io/traefik/reference/install-configuration/entrypoints)
In the [providers config](https://doc.traefik.io/traefik/reference/install-configuration/providers/overview/#supported-providers) we chose file:

```yaml
entryPoints:
  web:
    address: ":80"

providers:
  file:
    directory: "/root/traefik/dynamic"
```

In the root/dybamic config:
```bash
mkdir -p /root/traefik/dynamic
vim /root/traefik/dynamic/traefik.yaml
```

```yaml
http:
  routers:
    my-first-app:
      rule: "Host(`localhost`)"
      service: my-python-service
      entryPoints:
        - web

  services:
    my-python-service:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:8080"
```

We configure an [http router](https://doc.traefik.io/traefik/reference/routing-configuration/http/routing/router/#http-router) with a [rule](https://doc.traefik.io/traefik/reference/routing-configuration/http/routing/rules-and-priority/#host-and-hostregexp) that matches requests host set to domain localhost in entrypoint web.

Then a [service](https://doc.traefik.io/traefik/reference/routing-configuration/http/load-balancing/service) define how to distribute incoming traffic across your backend servers. In this case a **Service Load Balancer**: Routes traffic to backend servers using various load balancing strategies (*Weighted Round Robin is the Default strategy*). Then, [servers](https://doc.traefik.io/traefik/reference/routing-configuration/http/load-balancing/service/#servers) represent individual backend instances for your service.

```bash
cd /root/traefik
nohup traefik --configFile=traefik.yaml > traefik.log 2>&1 &
curl localhost
```

So, as a recap we launched an Ubuntu LXC container, installed Traefik binary, started a background Python app in port 8080 and configured Traefik:
- Install config: web port 80 as *entrypoint* and file *provider*
- Route config: file as well, with a *http router* with a single *rule* that routes based on localhost Host header to my python *service* with a single *instance*.

Things not correct or potential issues:

1. Using root user: fine for testing/local LXC
2. it won't work from host machine because the app binds to 127.0.0.1, need to bind app to 0.0.0.0 or container's ip or port forwarding from host.
3. `Host(localhost)` rule — Only matches requests with `Host: localhost header`. From another machine, use `Host(your-domain.com)` or remove host rule entirely.
4. No systemd/service setup so if container restarts, nothing auto starts
5. No firewall config, if exposing to network, need to allow ports

Middelware

Add a response header:

1. Edit the dynamic config file (`/root/traefik/dynamic/traefik.yaml`). Check [headers middleware](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/headers/)

```yaml
http:
  routers:
    my-first-app:
      rule: "Host(`localhost`)"
      service: my-first-service
      entryPoints:
        - web
      middlewares:
        - add-custom-header   # 👈 Add this line

  services:
    my-first-service:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:8080"

  # 👇 Add this middleware section
  middlewares:
    add-custom-header:
      headers:
        customResponseHeaders:
          X-Custom-Header: "Hello-from-Traefik"
```

2. Restart Traefik
```bash
pkill traefik
cd /root/traefik
nohup traefik --configFile=traefik.yaml > traefik.log 2>&1 &
```

3. Test
See if `X-Custom-Header: Hello-from-Traefik` is in the response headers:
```bash
curl -v localhost
```

Unfortunately there is no way to dry-run or test the configs.

Let's add the **docker provider** and let Traefik auto-discover containers via labels:

First, test using https://hub.docker.com/r/containous/whoami a tiny Go server that prints os information and HTTP request to output

```bash
docker run -d --name whoami1 containous/whoami
docker inspect whoami1 | grep IPAddress
curl <container's ip>
```

We cannot attach new labels to a running container. Labels are set at **container creation time and are immutable**. This is a Docker design choice.

You have two options:

Stop, remove, and recreate with new labels:

```bash
docker stop whoami1
docker rm whoami1
docker run -d \
  --name whoami1 \
  --label "traefik.enable=true" \
  --label "traefik.http.routers.whoami.rule=Host(\`whoami.local\`)" \
  containous/whoami
```
Then verify:
```bash
docker inspect whoami1 | grep -A2 "Labels"
```

Add the Docker provider to your install/static config

1. Edit /root/traefik/traefik.yaml:

```yaml
entryPoints:
  web:
    address: ":80"

providers:
  file:
    directory: "/root/traefik/dynamic"
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false  # only expose containers with traefik.enable=true
```
2. No need to change dynamic config.

The docker label tells Traefik to route requests with the Host header equal to whoami.local to this container.

Breakdown:

- `traefik.http.routers.whoami` → The router name is whoami
- `.rule` → Defines the routing condition
- `Host(whoami.local)` → Match when the HTTP Host header is exactly whoami.local

Kubernetes

```bash
mode=generated var_cpu="2" var_ram="4096" var_disk="20" bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/ubuntu.sh)"
```

```bash
apt update
apt install -y docker.io
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

k3d cluster create mycluster \
  --k3s-arg "--kubelet-arg=feature-gates=KubeletInUserNamespace=true@server:*"
```
The KubeletInUserNamespace feature gate tells kubelet "it's okay if we can't access /dev/kmsg, just keep running"

Install kubectl:
```bash
# Download the latest kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make it executable
chmod +x kubectl

# Move it to your PATH
sudo mv kubectl /usr/local/bin/

# Verify it works
kubectl version --client
```

```bash
kubectl get nodes

NAME                     STATUS   ROLES           AGE     VERSION
k3d-mycluster-server-0   Ready    control-plane   3m23s   v1.35.5+k3s1
```

```bash
# 1. Create deployment + service (one command!)
kubectl create deploy whoami --image=traefik/whoami && kubectl expose deploy whoami --port=80

deployment.apps/whoami created
service/whoami exposed

# 2. Port-forward (in background)
kubectl port-forward svc/whoami 8080:80 &

[1] 7412

# 3. Test it
curl localhost:8080

curl localhost:8080
Handling connection for 8080
Hostname: whoami-5cbdff98fc-zqkt6
IP: 127.0.0.1
IP: ::1
IP: 10.42.0.9
IP: fe80::4cc7:6eff:fe94:1a91
RemoteAddr: 127.0.0.1:49026
GET / HTTP/1.1
Host: localhost:8080
User-Agent: curl/8.5.0
Accept: */*
```

Traffic flow:

`curl → localhost:8080 → DIRECT to whoami pod (skips Traefik entirely)`

```bash
kubectl port-forward -n kube-system service/traefik 8081:80 &

kubectl run app1 --image=traefik/whoami --port=80 --expose
kubectl run app2 --image=nginx --port=80 --expose
kubectl run app3 --image=httpd --port=80 --expose

service/app1 created
pod/app1 created
service/app2 created
pod/app2 created
service/app3 created
pod/app3 created

# Create 3 Ingress rules
kubectl create ingress app1-ingress --rule="app1.local/*=app1:80"
kubectl create ingress app2-ingress --rule="app2.local/*=app2:80"
kubectl create ingress app3-ingress --rule="app3.local/*=app3:80"

ingress.networking.k8s.io/app1-ingress created
ingress.networking.k8s.io/app2-ingress created
ingress.networking.k8s.io/app3-ingress created

# ALL accessible through Traefik on the SAME port!
curl -H "Host: app1.local" localhost:8081  # → whoami
curl -H "Host: app2.local" localhost:8081  # → nginx
curl -H "Host: app3.local" localhost:8081  # → httpd

```
Without Traefik, we'd need 3 different ports.

```bash
kubectl describe pod -n kube-system -l app.kubernetes.io/name=traefik | grep "Image:"
    
    Image: rancher/mirrored-library-traefik:3.6.13

kubectl get deployment -n kube-system traefik -o yaml | grep -E "app.kubernetes.io/version|helm.sh/chart"
    
    helm.sh/chart: traefik-39.0.701_up39.0.7
```

### Traefik Static vs Dynamic Configuration (with k3d)

Traefik has two types of configuration:

- **Static Configuration** → how Traefik starts
- **Dynamic Configuration** → what Traefik routes

---

#### 1. Static Configuration

Static config is loaded when Traefik starts.

Source:

```text
Helm values.yaml
        |
        v
Kubernetes Deployment
        |
        v
Traefik startup args
```

Example:
```
--entryPoints.web.address=:8000/tcp
--entryPoints.websecure.address=:8443/tcp
--providers.kubernetescrd
```
Defines:
```
entryPoints
providers
metrics
dashboard/API
```
Example entryPoints:
```
web
 :8000  --> HTTP

websecure
 :8443  --> HTTPS
```

#### 2. Dynamic Configuration

Dynamic config is discovered while Traefik is running.

Defines:
```
routers
services
middlewares
TLS rules
```

Sources:

Docker:
```
Container labels
        |
        v
Docker provider
        |
        v
Traefik routes
```
Kubernetes:
```
Ingress / IngressRoute
        |
        v
Kubernetes API
        |
        v
Traefik provider
        |
        v
Traefik routes
```
Important:

`Ingress` and `IngressRoute` are different:

```
Ingress
   |
   v
--providers.kubernetesingress


IngressRoute
   |
   v
--providers.kubernetescrd
```

#### 3. Port-forward example

Command:

`kubectl port-forward -n kube-system service/traefik 8081:80`

Flow:
```
localhost:8081
        |
        v
Traefik Service :80
        |
        v
targetPort
        |
        v
Traefik container :8000
        |
        v
web entryPoint
```

Remember:

`Service port != container port`

The Service hides the internal Traefik port.

#### 4. Request flow

Command:

`curl -H "Host: app1.local" localhost:8081`

Flow:
```
Request
  |
  v
Traefik Service
  |
  v
web entryPoint
  |
  v
Router rule:
Host(app1.local)
  |
  v
Kubernetes Service
  |
  v
Pod
```

### Final comparison
Docker
```
Static:
Traefik args


Dynamic:
Container labels

Provider:
Docker API
```

Kubernetes
```
Static:
Helm values
Traefik args

Dynamic:
Ingress / IngressRoute

Provider:
Kubernetes API
```

The concept is the same:
```
Static config = how Traefik runs

Dynamic config = where traffic goes
```

Links:

- https://mogenius.com/blog-post/deploying-traefik-with-helm-for-simplified-kubernetes-ingress
- https://github.com/traefik/traefik-helm-chart/blob/v39.0.7/traefik/values.yaml
