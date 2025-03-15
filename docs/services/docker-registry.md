# Setting Up a Local Docker Registry

This guide covers how to set up a private Docker registry in your homelab, using your NAS for persistent storage. A local registry allows you to store and manage your Docker images without relying on Docker Hub or other public registries.

## Prerequisites

- Docker and Docker Compose installed on your VM
- NAS storage mounted (either via SMB or NFS)
- Basic understanding of Docker concepts

## Benefits of a Local Registry

- Store private images without uploading to public registries
- Faster image pulls for local deployments
- No rate limits or bandwidth constraints
- Complete control over your image storage

## Setting Up the Docker Registry

### 1. Create the Docker Compose File

Create a `docker-compose.registry.yml` file:

```bash
nano docker-compose.registry.yml
```

Add the following content:

```yaml
services:
  registry:
    image: registry:2
    container_name: docker-registry
    restart: always
    ports:
      - "5000:5000"
    volumes:
      - /mnt/prox-share-nfs/docker/volumes/registry:/var/lib/registry
    environment:
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
```

This configuration:
- Uses the official Docker registry image
- Maps port 5000 for registry access
- Stores registry data on your NAS mount
- Enables the ability to delete images from the registry

### 2. Prepare the Storage Directory

Create the directory on your NAS mount and set appropriate permissions:

```bash
# Create the directory for registry data
sudo mkdir -p /mnt/prox-share-nfs/docker/volumes/registry

# Set appropriate permissions
sudo chown 1000:1000 /mnt/prox-share-nfs/docker/volumes/registry
```

### 3. Deploy the Registry

Start the registry using Docker Compose:

```bash
docker compose -f docker-compose.registry.yml up -d
```

### 4. Verify the Registry is Running

Check that the registry container is running:

```bash
docker ps | grep registry
```

You should see the `docker-registry` container in the output.

## Using Your Local Registry

### Pushing Images to Your Registry

To push an image to your local registry:

1. Pull or build an image:
   ```bash
   # Pull a small test image
   docker pull hello-world
   ```

2. Tag it for your registry:
   ```bash
   # Tag it for your registry
   docker tag hello-world localhost:5000/hello-world
   ```

3. Push it to your registry:
   ```bash
   # Push it to your registry
   docker push localhost:5000/hello-world
   ```

### Listing Images in Your Registry

To see what images are stored in your registry:

```bash
curl -X GET http://localhost:5000/v2/_catalog
```

This should return something like:
```json
{"repositories":["hello-world"]}
```

To see the tags for a specific image:

```bash
curl -X GET http://localhost:5000/v2/hello-world/tags/list
```

### Pulling Images from Your Registry

To pull an image from your local registry:

```bash
docker pull localhost:5000/hello-world
```

## Accessing the Registry from Other Machines

### Option 1: Using IP Address

From other machines on your network, you can access the registry using the VM's IP address:

```bash
# Tag an image for your registry
docker tag myimage 192.168.1.100:5000/myimage

# Push to your registry
docker push 192.168.1.100:5000/myimage
```

### Option 2: Using DNS Name

If you have a local DNS server or have added entries to your hosts file:

```bash
# Tag an image for your registry
docker tag myimage registry.local:5000/myimage

# Push to your registry
docker push registry.local:5000/myimage
```

### Configuring Docker for Insecure Registry

By default, Docker requires HTTPS for registries. For a local registry without HTTPS, you need to configure Docker to allow insecure connections:

On each client machine, edit `/etc/docker/daemon.json`:

```bash
sudo nano /etc/docker/daemon.json
```

Add or modify:

```json
{
  "insecure-registries": ["localhost:5000", "<tailscale-ip>:5000"]
}
```

Make sure to replace `<tailscale-ip>` with your actual Tailscale IP address. Including both the local address and Tailscale IP ensures you can access the registry both locally and through your Tailscale network.

Restart Docker:

```bash
sudo systemctl restart docker
```

## Adding Security (Optional)

### Basic Authentication

To add basic authentication:

1. Create a password file:
   ```bash
   mkdir -p auth
   docker run --rm --entrypoint htpasswd httpd:2 -Bbn username password > auth/htpasswd
   ```

2. Update your Docker Compose file:
   ```yaml
   services:
     registry:
       image: registry:2
       container_name: docker-registry
       restart: always
       ports:
         - "5000:5000"
       volumes:
         - /mnt/prox-share-nfs/docker/volumes/registry:/var/lib/registry
         - ./auth:/auth
       environment:
         REGISTRY_STORAGE_DELETE_ENABLED: "true"
         REGISTRY_AUTH: htpasswd
         REGISTRY_AUTH_HTPASSWD_REALM: "Registry Realm"
         REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
   ```

3. Restart the registry:
   ```bash
   docker compose -f docker-compose.registry.yml up -d
   ```

4. Log in before pushing/pulling:
   ```bash
   docker login localhost:5000
   ```

## Maintenance

### Cleaning Up Unused Images

The Docker registry doesn't automatically garbage collect deleted images. To reclaim space:

```bash
# Run garbage collection
docker exec -it docker-registry bin/registry garbage-collect /etc/docker/registry/config.yml
```

### Backing Up the Registry

Since the registry data is stored on your NAS, it should be included in your regular NAS backup routine. However, you can also create a specific backup:

```bash
# Create a tar archive of the registry data
tar -czf registry-backup-$(date +%Y%m%d).tar.gz -C /mnt/prox-share-nfs/docker/volumes registry
```

## Troubleshooting

### Registry Container Won't Start

Check the logs:
```bash
docker logs docker-registry
```

### Can't Push Images

Verify the registry is running:
```bash
curl -X GET http://localhost:5000/v2/
```

Should return `{}`.

### Permission Issues

If you encounter permission issues with the volume:
```bash
# Check current ownership
ls -la /mnt/prox-share-nfs/docker/volumes/registry

# Fix permissions if needed
sudo chown -R 1000:1000 /mnt/prox-share-nfs/docker/volumes/registry
```

## Next Steps

- [Set up a Docker Compose environment](docker-compose-setup.md)
- [Configure automated builds with CI/CD](../maintenance/cicd-pipeline.md)
- [Implement container monitoring](../maintenance/monitoring.md) 