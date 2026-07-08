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

#### Middelware

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


### Docker 
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

### Kubernetes

Install helm from script:

https://helm.sh/docs/intro/install/#from-script
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```



```bash
k3d cluster create mycluster \
--k3s-arg "--kubelet-arg=feature-gates=KubeletInUserNamespace=true@server:*" \
--k3s-arg "--disable=traefik@server:0" \
--port "80:80@loadbalancer"
```
To verify Traefik is disabled:

`kubectl get pods -n kube-system | grep traefik`

```bash
helm version

version.BuildInfo{Version:"v4.2.2", GitCommit:"b05881cf967a5a09e19866799d0edfd40675803a", GitTreeState:"clean", GoVersion:"go1.26.4", KubeClientVersion:"v1.36"}

helm search repo traefik
NAME                	CHART VERSION	APP VERSION	DESCRIPTION
traefik/traefik     	41.0.1       	v3.7.5     	A Traefik based Kubernetes ingress controller
traefik/traefik-crds	1.18.0       	           	A Traefik based Kubernetes ingress controller
traefik/traefik-hub 	4.2.0        	v2.11.0    	Traefik Hub Ingress Controller
traefik/traefik-mesh	4.1.1        	v1.4.8     	Traefik Mesh - Simpler Service Mesh
traefik/traefikee   	4.2.8        	v2.12.8    	Traefik Enterprise is a unified cloud-native ne...
traefik/maesh       	2.1.2        	v1.3.2     	Maesh - Simpler Service Mesh
```

Ingress: no host rule (path-based routing)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - http:    # ← NO host field!
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```


From Traefik docs:
https://doc.traefik.io/traefik/getting-started/kubernetes/

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
kubectl create namespace traefik
```
Minimal `values.yaml`:
```yaml
# values.yaml
ports:
  web:
    port: 80
    # nodePort: 30000  # You can uncomment if you need a specific NodePort
    # No redirects, no HTTPS - just plain HTTP on port 80

api:
  dashboard: false   # Dashboard disabled for minimalism

# Explicitly enable the Kubernetes Ingress provider (this is the default, but let's be clear)
providers:
  kubernetesIngress:
    enabled: true
  # kubernetesGateway is NOT enabled here
```

```bash
helm install traefik traefik/traefik -n traefik -f values.yaml
NAME: traefik
LAST DEPLOYED: Wed Jul  1 22:26:13 2026
NAMESPACE: traefik
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
traefik with docker.io/traefik:v3.7.5 has been deployed successfully on traefik namespace!
```

`curl http://localhost` works

Static Configuration
The values.yaml file controls how Traefik starts up. With only the web entrypoint on port 80 and the Kubernetes Ingress provider enabled, the Helm chart supplies defaults for everything else. These settings become command-line arguments passed to the Traefik container when it starts.

Can check the args in the deployment with all available options https://doc.traefik.io/traefik/reference/install-configuration/configuration-options/#install-configuration-options


The resulting pod has entrypoints for web traffic on port 80, secure traffic on port 8443, metrics on port 9100, and the admin interface on port 8080. The pod also enables the ping healthcheck, turns on Prometheus metrics, disables the dashboard, and sets the log level to INFO.

Dynamic Configuration
The providers determine where Traefik looks for routing rules. The Kubernetes Ingress provider is enabled explicitly, and the Helm chart adds the Traefik CRD provider by default. Traefik watches the Kubernetes API for both standard Ingress resources and Traefik-specific resources like IngressRoute and Middleware. Traefik updates its routing live whenever these resources change, without needing a restart.

Kubernetes Resources
The Helm chart creates a Deployment that manages the Traefik pod. The pod contains the Traefik container with all the command-line arguments from your configuration. A Service of type LoadBalancer exposes Traefik to the cluster with ports 80 and 443, each with auto-assigned NodePorts. The k3d loadbalancer connects your Ubuntu LXC port 80 to this Service, completing the path from your host to your applications.

Traffic Flow
When you curl localhost from your Ubuntu LXC, the request reaches the k3d loadbalancer on port 80, which forwards it to the Traefik Service's NodePort. The Service routes it to the Traefik pod, where Traefik checks its dynamic configuration from the Ingress resources. It matches the request against your Ingress rules and forwards it to the appropriate backend service, ultimately reaching your application pod.

The configuration gives you a working Traefik setup because the Helm chart fills in all necessary defaults. The static configuration becomes pod arguments, the dynamic configuration comes from providers watching Kubernetes resources, and the k3d loadbalancer connects everything to your host.

Test:

```bash
kubectl run app1 --image=traefik/whoami --port=80 --expose
kubectl run app2 --image=nginx --port=80 --expose
kubectl run app3 --image=httpd --port=80 --expose

kubectl create ingress app1-ingress --rule="app1.local/*=app1:80"
kubectl create ingress app2-ingress --rule="app2.local/*=app2:80"
kubectl create ingress app3-ingress --rule="app3.local/*=app3:80"

# ALL accessible through Traefik on the SAME port!
curl -H "Host: app1.local" localhost  # → whoami
curl -H "Host: app2.local" localhost  # → nginx
curl -H "Host: app3.local" localhost  # → httpd

```

Links:

- https://mogenius.com/blog-post/deploying-traefik-with-helm-for-simplified-kubernetes-ingress
- https://github.com/traefik/traefik-helm-chart/blob/v39.0.7/traefik/values.yaml

### NEW Tests scenarios

Test all scenarios to see exactly how Traefik's `kubernetesIngress` provider behaves with different configurations.

```bash
# 1. Ingress with NO class (should work with Traefik)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app3-no-class
  namespace: default
  annotations:
    # No annotations at all
spec:
  # No ingressClassName specified
  rules:
  - host: app3-no-class.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app3
            port:
              number: 80
EOF

# 2. Ingress with nginx class (should NOT work with Traefik)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app3-nginx-class
  namespace: default
  annotations:
    # No annotations, just the class
spec:
  ingressClassName: nginx
  rules:
  - host: app3-nginx-class.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app3
            port:
              number: 80
EOF

# 3. Ingress with NO class but NGINX annotations (should work? Let's test)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app3-no-class-nginx-annos
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Test "no-class-nginx-anno";
spec:
  # No ingressClassName specified
  rules:
  - host: app3-no-class-nginx-annos.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app3
            port:
              number: 80
EOF

# 4. Ingress with nginx class AND nginx annotations (should NOT work)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app3-nginx-class-annos
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Test "nginx-class-anno";
spec:
  ingressClassName: nginx
  rules:
  - host: app3-nginx-class-annos.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app3
            port:
              number: 80
EOF

ingress.networking.k8s.io/app3-no-class created
ingress.networking.k8s.io/app3-nginx-class created
ingress.networking.k8s.io/app3-no-class-nginx-annos created
ingress.networking.k8s.io/app3-nginx-class-annos created
```

From logs `kubectl logs -n traefik deployment/traefik --tail=5`

```
2026-07-07T09:05:42Z INF Creating in-cluster Provider client providerName=kubernetescrd
2026-07-07T09:05:42Z INF Starting provider *traefik.Provider
2026-07-07T09:15:43Z WRN A new release of Traefik has been found: 3.7.6. Please consider updating.
2026-07-07T09:56:24Z INF Updated ingress status ingress=app3-no-class namespace=default
2026-07-07T09:56:24Z INF Updated ingress status ingress=app3-no-class-nginx-annos namespace=default`
```

Traefik only processed/updated the status for:
- app3-no-class
- app3-no-class-nginx-annos

It DID NOT process:

- app3-nginx-class
- app3-nginx-class-annos

This confirms that Traefik's kubernetesIngress provider ONLY watches ingresses with:
- No class specified (<none>) No Class = Works Regardless of Annotations 
- ingressClassName: traefik

### Test `kubernetesIngressNginx` provider:

Docs: https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress-nginx/

The kubernetesIngressNGINX provider:

✅ Reads NGINX-style Ingress resources (with ingressClassName: nginx)
✅ Translates NGINX annotations to Traefik routes

```bash
cat traefik-new-values.yaml
# Keep your existing settings
api:
  dashboard: false

ports:
  web:
    port: 80

# Add both providers (kubernetesIngress is enabled by default, but let's be explicit)
providers:
  kubernetesIngress:
    enabled: true
  kubernetesIngressNGINX:
    enabled: true
    watchNamespace: "default"
    ingressClass: "nginx"
    controllerClass: "k8s.io/ingress-nginx"
    watchIngressWithoutClass: false
    ingressClassByName: false

```

```bash
helm upgrade traefik traefik/traefik -f traefik-new-values.yaml -n traefik

Release "traefik" has been upgraded. Happy Helming!
NAME: traefik
LAST DEPLOYED: Tue Jul  7 12:47:38 2026
NAMESPACE: traefik
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
TEST SUITE: None
NOTES:
traefik with docker.io/traefik:v3.7.5 has been deployed successfully on traefik namespace!
```

In the docs in helm values it says `kubernetesIngressNginx` but
```bash
Error: UPGRADE FAILED: values don't meet the specifications of the schema(s) in the following chart(s):
traefik:
- at '/providers': additional properties 'kubernetesIngressNginx' not allowed
```
so it's `kubernetesIngressNGINX`. And:
```yaml
namespaces:
      - default`
```
```
Error: UPGRADE FAILED: values don't meet the specifications of the schema(s) in the following chart(s):
traefik:
- at '/providers/kubernetesIngressNGINX': additional properties 'namespaces' not allowed
```
it is:
```yaml
watchNamespace: "default"
```

`kubectl logs -n traefik deployment/traefik`

```bash
2026-07-07T10:56:24Z INF Creating in-cluster Provider client
2026-07-07T10:56:24Z INF Starting provider aggregator *aggregator.ProviderAggregator
2026-07-07T10:56:24Z INF Starting provider *traefik.Provider
2026-07-07T10:56:24Z INF Starting provider *ingressnginx.Provider
2026-07-07T10:56:24Z INF Starting provider *ingress.Provider
2026-07-07T10:56:24Z INF ingress label selector is: "" providerName=kubernetes
2026-07-07T10:56:24Z INF Creating in-cluster Provider client providerName=kubernetes
2026-07-07T10:56:24Z INF Starting provider *crd.Provider
2026-07-07T10:56:24Z INF label selector is: "" providerName=kubernetescrd
2026-07-07T10:56:24Z INF Creating in-cluster Provider client providerName=kubernetescrd
2026-07-07T10:56:24Z INF Starting provider *acme.ChallengeTLSALPN
2026-07-07T10:56:24Z ERR Invalid Ingress, skipping error="snippet annotations are not allowed when allowSnippetAnnotations is disabled" ingress=app3-nginx-class-annos namespace=default providerName=kubernetesingressnginx
```

#### Annotations

The problematic annotation is: `nginx.ingress.kubernetes.io/configuration-snippet`
The other annotations (`proxy-body-size` and `rewrite-target`) are not blocked. The error is only about `configuration-snippet`.


For now, just remove them. Option 1
```bash
# Remove only the configuration-snippet annotation
kubectl annotate ingress app3-nginx-class-annos nginx.ingress.kubernetes.io/configuration-snippet- --overwrite

# Now the ingress should work
curl -v -H "Host: app3-nginx-class-annos.local" localhost
```

```
┌─────────────────────────────────────────────────────────────┐
│                    Traefik (172.18.0.2)                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  kubernetesIngress Provider        kubernetesIngressNginx   │
│  ─────────────────────────        ──────────────────────    │
│  ✅ app1-ingress (traefik class)  ✅ app3-nginx-class       │
│  ✅ app2-ingress (traefik class)  ✅ app3-nginx-class-annos │
│  ✅ app3-ingress (traefik class)                            │
│  ✅ app3-no-class                                           │
│  ✅ app3-no-class-nginx-annos                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

The NGINX class ingresses (app3-nginx-class and app3-nginx-class-annos) still have no ADDRESS even though they work. This is because the NGINX provider needs a publishService configuration to know which service's IP to use for the status.

```bash
k get ingress -A
NAMESPACE   NAME                        CLASS     HOSTS                             ADDRESS      PORTS   AGE
default     app1-ingress                traefik   app1.local                        172.18.0.2   80      5d13h
default     app2-ingress                traefik   app2.local                        172.18.0.2   80      5d13h
default     app3-ingress                traefik   app3.local                        172.18.0.2   80      5d13h
default     app3-nginx-class            nginx     app3-nginx-class.local                         80      92m
default     app3-nginx-class-annos      nginx     app3-nginx-class-annos.local                   80      92m
default     app3-no-class               traefik   app3-no-class.local               172.18.0.2   80      92m
default     app3-no-class-nginx-annos   traefik   app3-no-class-nginx-annos.local   172.18.0.2   80      92m
```

The NGINX provider doesn't automatically know which LoadBalancer IP to publish. It needs to be told explicitly which service to use for the status address. The Fix: Set publishService. But as doc says:

https://doc.traefik.io/traefik/migrate/nginx-to-traefik/#step-1-install-traefik-alongside-nginx

```
Running both controllers against the same Ingresses creates contention on the status.loadBalancer.ingress[] field. Before installing, review the Ingress Status Race Condition section in Step 3 and decide which mitigation to apply (disable publishService on Traefik, or use a transitional IngressClass).
```

Why This Happens
Both providers are watching and reconciling the same Ingress resources. When they do, they both try to write the IP address of their respective LoadBalancer service into the status field. They constantly overwrite each other, leading to a flapping status.

Crucially, this does not affect routing. Your curl commands work perfectly because the routing is handled internally by Traefik, independent of the status field.


Read more https://doc.traefik.io/traefik/migrate/nginx-to-traefik/#status-race


The NGINX Controller is no longer needed because:

- Traefik reads ingressClassName: nginx ingresses
- Traefik translates NGINX annotations
- Your apps work exactly as before

### Moving to Gateway API

Gateway API is the next-generation Kubernetes Ingress API, offering:

- More expressive routing (HTTP header matching, weight-based traffic splitting)
- Role-based access control (different teams can manage different parts)
- Better separation of concerns (infrastructure vs application owners)
- Cross-namespace routing (secure and flexible)

Traefik supports all three approaches simultaneously, so you can migrate gradually:

```yaml
# Traefik can run ALL providers at the same time:
providers:
  # Current providers (keep these running)
  kubernetesIngress:
    enabled: true
  
  kubernetesIngressNGINX:
    enabled: true
  
  # Future Gateway API provider
  kubernetesGateway:
    enabled: true  # ← Add this when ready
```

#### Step 1: Install Gateway API CRDs (if not already present)

```bash
# Install the Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

#### Step 2: Enable Gateway Provider in Traefik
```bash
# Update Traefik configuration
cat <<EOF > traefik-with-gateway.yaml
api:
  dashboard: false

ports:
  web:
    port: 80
  gateway:
    port: 8000
    exposedPort: 8000
    protocol: TCP

providers:
  kubernetesIngress:
    enabled: true
  kubernetesIngressNGINX:
    enabled: true
  kubernetesGateway:
    enabled: true
EOF

helm upgrade traefik traefik/traefik -f traefik-with-gateway.yaml -n traefik
```

Error:

```bash
helm upgrade traefik traefik/traefik -f traefik-with-gateway.yaml -n traefik

Error: UPGRADE FAILED: execution error at (traefik/templates/gateway.yaml:39:12): ERROR: port 8000 is not declared in ports
```
Fix: add port gateway on port 8000

A warning is given after `helm apply`:

```bash
Release "traefik" has been upgraded. Happy Helming!
NAME: traefik
LAST DEPLOYED: Wed Jul  8 10:39:26 2026
NAMESPACE: traefik
STATUS: deployed
REVISION: 3
DESCRIPTION: Upgrade complete
TEST SUITE: None
NOTES:
traefik with docker.io/traefik:v3.7.5 has been deployed successfully on traefik namespace!

⚠️ DEPRECATION WARNING: Gateway API CRDs will no longer be shipped with this chart in a future major version.
You will need to install them yourself before deploying Traefik v3.7:
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml
```

More errors, fix by:
```bash
helm upgrade traefik traefik/traefik -f traefik-with-gateway.yaml -n traefik

Release "traefik" has been upgraded. Happy Helming!
NAME: traefik
LAST DEPLOYED: Wed Jul  8 10:53:26 2026
NAMESPACE: traefik
STATUS: deployed
REVISION: 4
DESCRIPTION: Upgrade complete
TEST SUITE: None
NOTES:
traefik with docker.io/traefik:v3.7.5 has been deployed successfully on traefik namespace!

⚠️ DEPRECATION WARNING: Gateway API CRDs will no longer be shipped with this chart in a future major version.
You will need to install them yourself before deploying Traefik v3.7:
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml


cat traefik-with-gateway.yaml
api:
  dashboard: false

ports:
  web:
    port: 80
  gateway:
    port: 8000
    exposedPort: 8000
    protocol: TCP

providers:
  kubernetesIngress:
    enabled: true
  kubernetesIngressNGINX:
    enabled: true
  kubernetesGateway:
    enabled: true
    experimentalChannel: false
```

In the traefik logs:

```
E0708 08:54:11.348865       1 reflector.go:204] "Failed to watch" err="failed to list *v1.TLSRoute: the server could not find the requested resource (get tlsroutes.gateway.networking.k8s.io)" logger="UnhandledError" reflector="k8s.io/client-go@v0.35.2/tools/cache/reflector.go:289" type="*v1.TLSRoute"
E0708 08:54:25.031079       1 reflector.go:204] "Failed to watch" err="failed to list *v1.BackendTLSPolicy: the server could not find the requested resource (get backendtlspolicies.gateway.networking.k8s.io)" logger="UnhandledError" reflector="k8s.io/client-go@v0.35.2/tools/cache/reflector.go:289" type="*v1.BackendTLSPolicy"
E0708 08:55:03.400242       1 reflector.go:204] "Failed to watch" err="failed to list *v1.BackendTLSPolicy: the server could not find the requested resource (get backendtlspolicies.gateway.networking.k8s.io)" logger="UnhandledError" reflector="k8s.io/client-go@v0.35.2/tools/cache/reflector.go:289" type="*v1.BackendTLSPolicy"
E0708 08:55:08.449929       1 reflector.go:204] "Failed to watch" err="failed to list *v1.TLSRoute: the server could not find the requested resource (get tlsroutes.gateway.networking.k8s.io)" logger="UnhandledError" reflector="k8s.io/client-go@v0.35.2/tools/cache/reflector.go:289" type="*v1.TLSRoute"
```

Install experimental CRDs test:

```bash
# Install experimental Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/experimental-install.yaml

# Check if errors stop
kubectl logs -n traefik deployment/traefik --tail=20 -f
```

#### Step 3: Create GatewayClass and Gateway

```bash
# Create a GatewayClass (like an IngressClass for Gateway API)
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: traefik
spec:
  controllerName: traefik.io/gateway-controller
---
# Create a Gateway (like a LoadBalancer service)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: traefik-gateway
  namespace: default
spec:
  gatewayClassName: traefik
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
EOF
```

#### Step 4: Create Gateway API Routes (Gradual Migration)

Create Gateway API resources while keeping your existing ingresses:

```yaml
# Example: Migrate app1 from Ingress to Gateway API
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app1-route
  namespace: default
spec:
  parentRefs:
  - name: traefik-gateway
    namespace: default
  hostnames:
  - "app1.local"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app1
      port: 80
```

```bash
# Test app1.local - should work exactly as before
curl -H "Host: app1.local" localhost

# Check Traefik logs to confirm it's using the Gateway route
kubectl logs -n traefik deployment/traefik --tail=50 | grep -i "app1"
```

Delete the Old Ingress
Once you've confirmed the HTTPRoute works, then delete the Ingress:

```bash
# Delete the old Ingress for app1
kubectl delete ingress app1-ingress


# Verify app1.local still works
curl -H "Host: app1.local" localhost
```

Result:
```bash
k get ingress -A
NAMESPACE   NAME                        CLASS     HOSTS                             ADDRESS      PORTS   AGE
default     app2-ingress                traefik   app2.local                        172.18.0.2   80      6d11h
default     app3-ingress                traefik   app3.local                        172.18.0.2   80      6d11h
default     app3-nginx-class            nginx     app3-nginx-class.local                         80      23h
default     app3-nginx-class-annos      nginx     app3-nginx-class-annos.local                   80      23h
default     app3-no-class               traefik   app3-no-class.local               172.18.0.2   80      23h
default     app3-no-class-nginx-annos   traefik   app3-no-class-nginx-annos.local   172.18.0.2   80      23h


k get httproute -A
NAMESPACE   NAME         HOSTNAMES        AGE
default     app1-route   ["app1.local"]   103s


curl -H "Host: app1.local" localhost
Hostname: app1
IP: 127.0.0.1
IP: ::1
IP: 10.42.0.25
IP: fe80::ec43:e0ff:fe02:c4a8
RemoteAddr: 10.42.0.30:47840
GET / HTTP/1.1
Host: app1.local
User-Agent: curl/8.5.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 10.42.0.1
X-Forwarded-Host: app1.local
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik-8497f8876f-f99x2
X-Real-Ip: 10.42.0.1
```

Traefik is handling both:

- Traditional Ingress resources (app2.local, app3.local)
- Gateway API HTTPRoute (app1.local)

Both are being routed through the same Traefik instance.

## Important: 

`Create HTTPRoute FIRST → Test it → Then delete Ingress`