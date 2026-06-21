# GKE Deployment Playbook (Plain-English Edition)

A step-by-step guide to put **any** containerized web app on Google Kubernetes
Engine (GKE), give it a real load balancer and automatic HTTPS, and then delete
it cleanly so you stop paying.

Every step explains **why** it's there — so you actually understand what's going
on instead of just copying commands.

> Tested on the **Unyt** project (Next.js + MongoDB Atlas + Upstash Redis).
> Written so your next project is just "change the values at the top and run."

---

## How it all fits together (read this once)

When someone opens your site, their request travels through a chain of pieces.
Each piece has exactly one job:

```
Browser
  │  https://your-domain
  ▼
DNS (Cloudflare)            → turns your domain name into an IP address
  ▼
Cloud Load Balancer        → one stable public "front door"; spreads traffic
  │  (created automatically by ingress-nginx)
  ▼
ingress-nginx              → decides where each request goes; handles HTTPS
  │
  ▼
Service                    → a stable internal address for a group of pods
  │  (shares traffic across them)
  ▼
Pods (your app ×N)         → the actual running containers, across machines
  │
  ├── Managed DB (Atlas)   → your data lives HERE, not in the cluster
  └── Managed cache (Upstash)
```

**Why this shape?**

| Piece | What it does | Why we use it |
|---|---|---|
| **GKE (managed Kubernetes)** | Google runs the "brain" of the cluster for you | You don't have to maintain the control plane yourself, and nodes spread across zones so it stays up if one fails. A *zonal* cluster's brain is **free**, and new accounts get **$300 credit** — the cheapest way to learn real Kubernetes. |
| **ingress-nginx** | One load balancer that routes traffic to all your apps | It's the standard, and it works the same on any cloud (AWS, Azure, bare metal). Installing it **automatically creates the cloud load balancer** for you. |
| **cert-manager** | Gets and auto-renews free HTTPS certificates | Free HTTPS that renews itself — no more "the certificate expired" outages. |
| **Managed DB / cache** | Your data lives **outside** the cluster | Running databases inside Kubernetes is hard (backups, failover, storage). Keeping data outside means the cluster holds nothing important. |

**The golden rule this gives you:** the cluster holds **no data**, so tearing it
down loses nothing. That's why Step 10 (delete everything) is completely safe.

---

## Before you start

**Your project should already have:**
- a `Dockerfile` (builds a runnable image),
- a `k8s/` folder with `deployment.yaml`, `service.yaml`, and `hpa.yaml`,
- a `.env` file with your app's secret values.

**Installed on your machine:** `gcloud`, `kubectl`, `helm`,
`gke-gcloud-auth-plugin`, `docker` (with buildx).

**Accounts you need:** Google Cloud (with billing linked — the $300 free trial
counts), a DNS provider like Cloudflare, and a Docker Hub account.

---

## Step 0 — Set your variables (once per project)

Everything below uses these. Set them once here, and the rest of the guide just
works.

```bash
export APP=myapp                              # the name your k8s files use
export NAMESPACE=myapp
export IMAGE=dockerhubuser/myapp              # your Docker Hub repo
export TAG=1.0.0
export DOMAIN=app.example.com                 # your public web address
export GCP_PROJECT=my-gcp-project
export ZONE=europe-west2-a                    # pick a region near your users
export LE_EMAIL=you@example.com               # for certificate expiry notices
```

> Tip: your files in `k8s/` already contain real names (like `unyt-app`). Just
> make sure `$APP` matches whatever name those files use — keep them consistent.

---

## Step 1 — Build and push your image (it must be **amd64**)

```bash
docker login -u dockerhubuser
docker buildx create --name xbuilder --use --bootstrap 2>/dev/null || docker buildx use xbuilder

docker buildx build --platform linux/amd64 \
  --build-arg NEXT_PUBLIC_BASE_URL="https://$DOMAIN" \
  -t $IMAGE:$TAG --push .
```

**Why `--platform linux/amd64`?** GKE machines run on Intel/AMD chips. If you
build on an Apple-Silicon (M-series) Mac, a normal build makes an ARM image and
your pods crash with `exec format error`. This flag forces the right chip type.

**Why pass `NEXT_PUBLIC_*` values as build-args?** In Next.js, anything starting
with `NEXT_PUBLIC_` is **baked into the code when you build it**, not read later.
So those values have to be present *right now*, during the build. (You'd add one
`--build-arg` line per public variable your app uses.)

**Why bump `$TAG` every time?** If you always reuse the same tag (like `latest`),
you can never be sure which version is actually running, and caches may serve old
images. A fresh tag each time means you always know exactly what's deployed.

---

## Step 2 — Create the cluster

```bash
gcloud config set project $GCP_PROJECT
gcloud config set compute/zone $ZONE
gcloud services enable container.googleapis.com      # needs billing linked

gcloud container clusters create $APP \
  --zone $ZONE --num-nodes 2 --machine-type e2-small

gcloud container clusters get-credentials $APP --zone $ZONE
kubectl get nodes -o wide        # you should see 2 nodes marked "Ready"
```

**Why a *zonal* cluster (`--zone`, not `--region`)?** A zonal cluster's control
plane is free. A regional one runs three control planes and costs more. For
learning and small apps, zonal is the cheap, sensible choice.

**Why 2 nodes?** So your app's copies (pods) can spread across two machines, and
you can actually *see* high availability in action.

**Why `get-credentials`?** It saves this cluster's connection details so that
`kubectl` knows to talk to **this** cluster.

---

## Step 3 — Namespace + secrets (the simple way)

The idea here: open your terminal, go into your project folder, write your
secrets into one file, and apply it. That's it.

**1. Go to your project and create the namespace:**

```bash
cd ~/path/to/your-project        # go to your project folder
kubectl create namespace $NAMESPACE
```

**2. Create a file called `secrets.yaml`** in your project, and copy each value
from your `.env` into it like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets          # must match the name your deployment.yaml expects
  namespace: myapp
type: Opaque
stringData:
  MONGODB_URI: "mongodb+srv://user:pass@cluster.mongodb.net/mydb"
  REDIS_URL: "https://your-upstash-url"
  SOME_API_KEY: "abc123"
  # ...add one line for every variable in your .env
```

**3. Apply it:**

```bash
kubectl apply -f secrets.yaml
kubectl get secret myapp-secrets -n $NAMESPACE      # check it shows up
```

**Why this is easy:** the `stringData:` section lets you paste your real values
as **plain text** — Kubernetes encodes them for you. (You may see other examples
that use `data:` instead; those need every value pre-encoded in base64, which is
extra hassle. `stringData` skips all that.)

**⚠️ One important thing — never commit this file to Git.** Your `secrets.yaml`
holds real passwords in plain text. The trade-off for this being simple is that
the plaintext file lives on your laptop, so keep it there and never let it reach
version control. Add it to `.gitignore` right now:

```bash
echo "secrets.yaml" >> .gitignore
```

**Why a namespace?** Think of it as a folder that holds everything for this app.
It keeps things tidy and makes cleanup trivial later — delete the folder and
everything inside it goes with it.

> **Managed DB allowlist:** your cluster connects to your database from Google's
> IP addresses, not your laptop. In Atlas → Network Access, allow the cluster's
> outgoing IP (or `0.0.0.0/0` for a short-lived test). HTTPS-only services like
> Upstash don't need this.

---

## Step 4 — Deploy the app

```bash
kubectl apply -f k8s/service.yaml -f k8s/hpa.yaml
kubectl apply -f k8s/deployment.yaml
kubectl -n $NAMESPACE rollout status deploy/$APP
kubectl -n $NAMESPACE get pods -o wide          # pods should land on both nodes
```

**Why apply the Service together with the Deployment?** The Service is a stable
internal address that always points at your healthy pods, even when they restart
or scale up and down. Other things (like the ingress later) talk to the Service,
not to individual pods.

**Why the HPA?** The HorizontalPodAutoscaler automatically adds or removes copies
of your app based on CPU usage — so it grows under load and shrinks when quiet,
instead of being a fixed size.

> **Watch out — tiny machines can't fit big requests.** An `e2-small` only has
> about 940m of usable CPU, and Kubernetes' own background services eat most of
> it. If two pods each *reserve* 250m of CPU, they won't both fit, and one stays
> stuck as `Pending`. (Scheduling is based on what a pod **reserves**, not what
> it actually uses.) On small machines, lower the reservation:
> ```bash
> kubectl -n $NAMESPACE set resources deployment/$APP --requests=cpu=100m,memory=192Mi
> ```

**Quick internal test before exposing it to the world:**

```bash
kubectl -n $NAMESPACE run curltest --image=curlimages/curl --restart=Never --rm -i --quiet -- \
  -s -o /dev/null -w "HTTP %{http_code}\n" http://${APP}-service/
```

---

## Step 5 — Expose it: install ingress-nginx (this creates the load balancer)

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace

kubectl -n ingress-nginx get svc ingress-nginx-controller -w   # wait for EXTERNAL-IP
```

**Why does this create a load balancer?** ingress-nginx's own Service is set to
`type: LoadBalancer`. On GKE, that tells Google to automatically create a real
cloud load balancer and give it a public IP address. That IP is your single
public front door.

**Why use an ingress controller at all (instead of exposing the app directly)?**
One load balancer can serve **many** apps, route by web address, and handle HTTPS
in one place. Without it, you'd pay for and manage a separate load balancer for
every single service.

---

## Step 6 — Point your domain at the load balancer

In Cloudflare (or your DNS provider), add an **A** record: `$DOMAIN` → the
`EXTERNAL-IP` from Step 5. Set it to **DNS only / grey cloud**.

```bash
dig +short $DOMAIN @1.1.1.1      # should return the EXTERNAL-IP
```

**Why "grey cloud" (DNS-only) for now?** In a moment, cert-manager proves you own
the domain by answering a challenge *at that IP*. If Cloudflare is proxying the
traffic (orange cloud), it gets in the way and the challenge can fail. Get the
certificate working with grey cloud first, then switch to orange later if you
want Cloudflare's CDN and firewall in front.

---

## Step 7 — Automatic HTTPS with cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
kubectl -n cert-manager rollout status deploy/cert-manager
```

Now create the **issuer** (who signs the certificate) and the **Ingress** (the
routing rule that also requests the certificate):

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: $LE_EMAIL
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${APP}-ingress
  namespace: $NAMESPACE
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # ← this triggers the certificate
spec:
  ingressClassName: nginx
  tls:
    - hosts: [$DOMAIN]
      secretName: ${APP}-tls                            # certificate gets stored here
  rules:
    - host: $DOMAIN
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ${APP}-service
                port:
                  number: 80
EOF

kubectl -n $NAMESPACE get certificate -w     # wait for READY=True (about 30s–2min)
```

**What's happening here:** the annotation on the Ingress tells cert-manager "get
me a certificate for these addresses." cert-manager asks Let's Encrypt, proves
you own the domain (through a challenge served by ingress-nginx), receives the
certificate, and stores it in the `${APP}-tls` secret. ingress-nginx then serves
HTTPS using it — and cert-manager renews it automatically before it expires. No
manual certificate work, ever.

---

## Step 8 — Check it actually works

```bash
curl -I https://$DOMAIN                       # expect HTTP/2 200
echo | openssl s_client -connect $DOMAIN:443 -servername $DOMAIN 2>/dev/null \
  | openssl x509 -noout -issuer -dates        # issuer should say Let's Encrypt
curl -sI http://$DOMAIN | grep -i location    # plain http should redirect (308) to https
```

---

## Step 9 — Ship a new version

```bash
# build & push a new TAG (Step 1), then:
kubectl -n $NAMESPACE set image deployment/$APP $APP=$IMAGE:$TAG
kubectl -n $NAMESPACE rollout status deployment/$APP
kubectl -n $NAMESPACE rollout undo deployment/$APP     # instant rollback if needed
```

**Why `set image` + `rollout`?** Kubernetes does a rolling update — new pods
start and pass their health checks *before* the old ones are removed, so there's
no downtime. If the new version is broken, `rollout undo` snaps you back to the
previous one.

> **Health checks must match the image.** If your file's liveness/readiness check
> points at `/api/health`, the image has to actually serve that path — otherwise
> pods never become "Ready" and the rollout gets stuck. Deploying an older image
> that doesn't have it? Temporarily point the checks at `/` instead.

---

## Step 10 — DELETE EVERYTHING (this is real money / credit)

A running cluster and load balancer charge you **every hour**, even when idle.
Because all your data lives in managed services (Atlas / Upstash), the cluster is
disposable — deleting it loses nothing.

```bash
# 1. Remove the thing that created the load balancer FIRST, so Google releases it.
#    (If you delete the cluster first, the load balancer can get orphaned
#     and keep billing you forever.)
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace $NAMESPACE

# 2. Delete the cluster (machines + control plane).
gcloud container clusters delete $APP --zone $ZONE --quiet

# 3. Clean up locally.
docker buildx rm xbuilder
```

**Double-check nothing is still billing you:**

```bash
gcloud container clusters list            # should be empty
gcloud compute forwarding-rules list      # should be empty (these ARE the load balancers)
gcloud compute addresses list             # release any reserved IPs you don't need
```

Finally, delete the `$DOMAIN` DNS record (it now points at a dead IP).
**Orphaned load balancers and reserved IPs are the #1 cause of surprise "why am I
still being charged?" bills** — the `forwarding-rules` check above is the one that
catches them.

---

## Quick FAQ (for next time)

- **Why GKE over EKS / AKS?** It's the cheapest to learn on: free zonal control
  plane plus $300 credit. AWS's EKS charges about $73/month for the control plane
  no matter what. The skills carry over to all three — only the cluster *setup*
  differs.
- **Why ingress-nginx over a cloud's native option or Gateway API?** It's portable
  (works the same on any cloud) and it's the most common one out there. Gateway
  API is the modern successor worth learning next, but ingress-nginx is what most
  jobs expect *today*.
- **Why not just use Vercel / Render / Fly?** Those are great for shipping fast —
  but they hide the infrastructure. This stack is for *learning* the real
  production machinery (load balancers, ingress, HTTPS, scaling) that those
  platforms hide from you.
- **Why a managed database instead of running one in the cluster?** Databases
  need backups, failover, and persistent storage — hard to get right. Keeping
  data outside makes the cluster safe to destroy and recreate whenever you want.