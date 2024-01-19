# Kemkas k8s Deployment

Currently running on managed Kubernetes and managed PostgreDB (with a staging enviroonemnt on a managed App Platform)

[![DigitalOcean Referral Badge](https://web-platforms.sfo2.cdn.digitaloceanspaces.com/WWW/Badge%203.svg)](https://www.digitalocean.com/?refcode=ff66274a0cf1&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)

Note: the above button is a referal link.

## ToC
- [Argo CD](#argo-cd)
- [1password ESO connect server](#1password-connect-server)
- [Kemkas](#kemkas)

## Argo CD

Goal: continous deployment automation with rollback capability

Files:
- [argocd-ingress.yml](./argocd/argocd-ingress.yaml) for connecting the service with the load balancer and nginx reverse proxy
- [argocd-values.yml](./argocd/argocd-values.yaml) config for the argocd helm chart to use SSL termination at NGINX.
- [cert-manager-issuer.yaml](./argocd/cert-manager-issuer.yaml) SSL certificate config for the argocd.kemkas.hu domain

TODO: 
- [ ] gRPC setup for ArgoCD to use the CLI. Workaround: k8s port proxy to localhost.

Setup instructions

1. DO 1-click install argoCD into k8s cluster (TODO: replace with helm install command, I suspect behind the scenes the 1-click install does the same)
1. DO 1-click install ingress-nginx into k8s cluster (TODO: replace with helm install command, I suspect behind the scenes the 1-click install does the same)
1. DO 1-click install cert-manager into k8s cluster (TODO: replace with helm install command, I suspect behind the scenes the 1-click install does the same)
1. helm upgrade ingress-nginx to fix cert-manager pod2pod communication and enable proxy protocoll `helm upgrade ingress-nginx ingress-nginx/ingress-nginx --version 4.8.2 --namespace ingress-nginx --values ingress-nginx/values.yaml`
1. Create DNS A record for argocd.kemkas.hu pointing to the Load Balancer IP address (Load balancer is a DO object created with the ingress-nginx helm chart install)
1. helm update argocd with `helm upgrade argocd argo/argo-cd --version 4.9.4 --namespace argocd -f argocd-values.yaml`
1. Apply all files in `argocd` directory
    ```
    kubectl apply -f cert-manager-issuer.yaml
    kubectl apply -f argocd-ingress.yaml
    ```

## 1Password Connect Server

Goal: Store secrets in 1password instead of kubernetes, so we can push all kubernetes config while keeping the secrets safe in 1password

Files:
- [1password-secret-store.yaml](./1password-connect-server/1password-secret-store.yaml): kuberentes ESO Secret Store to be referenced by external secrets
- [connect-server-access-token-secret.yaml](./1password-connect-server/connect-server-access-token-secret.yaml): kubernetes secret file to connect with the 1password connect server without the actual secret token. the token can be generated on 1password website or CLI.
- [docker-compose.yaml](./1password-connect-server/docker-compose.yaml): docker compose config to run the 1password connect server on a separate machine. Requires a `1password-credentials.json` file locally in the same directory, which can be obtained from the 1password website or CLI.
- [connect-server.conf](./1password-connect-server/connect-server.conf) nginx config for the SSL terminating proxy for the 1password connect server (so that secrets don't travel unencrypted)

TODO: 
- [ ] explore running the 1password connect server on the same k8s cluster to simplify infrastructure
- [ ] write script to automate the setup instraction (using `doctl` for full infra automation)

Setup:
1. create DO droplet (with docker template)
1. create DNS A record pointing to the droplet
1. install nginx, certbot `sudo apt install nginx certbot python3-certbot-nginx`
1. copy `docker-compose.yaml` and `1password-credentials.json` to the droplet and run `docker compose up -d`
1. copy connect-server nginx config to `/etc/nginx/sites-available` and link it to sites enabled `ln -s /etc/nginx/sites-available/connect-server.conf /etc/nginx/sites-enabled/`
1. test nginx config `nginx -t`
1. reload nginx config `nginx -s reload`
1. create SSL certificate with certbot `certbot --nginx -d 1password-connect.dev.kemkas.hu` (email address and accepting terms is required!)
1. setup firewall to block TCP/8080 and TCP/8081 to avoid unencrypted access to the connect server (allow TCP/443 and TCP/22)
1. helm install the external secret operator `helm repo add external-secrets https://charts.external-secrets.io` and `helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace --set installCRDs=false` (from [https://external-secrets.io/](https://external-secrets.io/))
1. create namespace for secret store `kubectl create ns kemkas`
1. create connect server access token secret in k8s `kubectl apply -f ./1password-connect-server/connect-server-access-token-secret.yaml`
1. create secret store `kubectl apply -f ./1password-connect-server/1password-secret-store.yaml`

## Kemkas

Goal: deploy an existing docker image into the k8s cluster with proper secrets and expose it on HTTPS with proper SSL cert

Files:
- [kemkas-external-secrets.yaml](./kemkas/kemkas-external-secrets.yaml): syncs external secrets from the 1password secret store
- [kemkas-deployment.yaml](./kemkas/kemkas-deployment.yaml): configures the kemkas server docker image to run on the cluster
- [kemkas-service.yaml](./kemkas/kemkas-service.yaml): exposes the kemkas server deployment to the cluster
- [kemkas-ssl-cert-issuer.yaml](./kemkas/kemkas-ssl-cert-issuer.yaml): config for issuing SSL certs for accessing the kemkas server on the internet
- [kemkas-ingress.yaml](./kemkas/kemkas-ingress.yaml): exposes the kemkas server on the internet through the nginx ingress proxy

Setup:
1. connect Digital Ocean Container Registry to the k8s cluster `doctl registry kubernetes-manifest | kubectl apply -f -`
1. create external secrets for the upcoming kemkas deployment `kubectl apply -f ./kemkas/kemkas-external-secrets.yaml`
1. create kemkas deployment `kubectl apply -f ./kemkas/kemkas-deployment.yaml`
1. create service to export the kemkas deployment `kubectl apply -f ./kemkas/kemkas-service.yaml`
1. create SSL cert issuer `kubectl apply -f ./kemkas/kemkas-ssl-cert-issuer.yaml`
1. create ingress for the kemkas service `kubectl apply -f ./kemkas/kemkas-ingress.yaml`
