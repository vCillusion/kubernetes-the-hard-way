# Setup the Kubernetes Docker Registry
The guide to setup the Kubernetes Docker registry

## Create certificates

```
mkdir registry
cd registry
mkdir certs
openssl req \
  -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key \
  -addext "subjectAltName = DNS:reglocal" \
  -x509 -days 365 -out certs/domain.crt
```

Copy the domain.crt file to /etc/docker/certs.d/reglocal:5000/ca.crt on every Docker host. You do not need to restart Docker.
```
sudo mkdir -p /etc/docker/certs.d/reglocal:5000
sudo cp /home/vagrant/registry/certs/domain.crt /etc/docker/certs.d/reglocal:5000/ca.crt
```
Reference: https://docs.docker.com/registry/insecure/

## make auth

```
mkdir auth
docker run \
  --entrypoint htpasswd \
  httpd:2 -Bbn testuser testpassword > auth/htpasswd
```
Reference: https://docs.docker.com/registry/deploying/#get-a-certificate

## start docker registry
```
docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name reglocal \
  -v "$(pwd)"/data:/var/lib/registry \
  -v "$(pwd)"/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v "$(pwd)"/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

## add entry to the hosts file and perform docker login
```
sudo vi /etc/hosts
192.168.5.11 reglocal
docker login reglocal:5000
```
> output
```
Authenticating with existing credentials...
WARNING! Your password will be stored unencrypted in /home/vagrant/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

# access from kubernetes
## add certs and edit hosts file for all workers
```
for x in worker-1 worker-2; do scp certs/domain.crt ${x}:~/ ; done
```

for worker-1 and worker-2, execute below
```
sudo mkdir -p /etc/docker/certs.d/reglocal:5000
sudo cp domain.crt /etc/docker/certs.d/reglocal:5000/ca.crt
```

## add secret to kubernetes for docker registry
```
cat ~/.docker/config.json
kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=/home/vagrant/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson

kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```
References:
https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account
https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/

