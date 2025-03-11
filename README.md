# Docker Setups

## Table of Contents
1. [Docker Private Repository](#docker-private-repository)
2. [Before Pushing Images](#before-pushing-images)
3. [Multi Stage Build](#multi-stage-build)

## Docker Private Repository

A private Docker repository allows you to store and distribute Docker images securely within your infrastructure. Follow the steps below to set up a private Docker registry.

### Prerequisites
- Installed Docker on your system.  
- Sufficient disk space to store images.  
- A network-accessible host to serve the registry.  

### Step 1: Start a Local Docker Registry
Run the following command to start a basic, insecure local registry on port 5000:

```sh
docker run -d -p 5050:5000 --restart unless-stopped --name local-docker-registry  -v /mnt/sdb2-partition/dockerImages:/var/lib/registry -e REGISTRY_STORAGE_DELETE_ENABLED=true registry:2
```

### Breakdown

1. **`docker run`**  
   This command is used to create and start a new Docker container.

2. **`-d` (detached mode)**  
   Runs the container in the background so that it doesn’t block the terminal.

3. **`-p 5050:5000`**  
   - Maps port **5000** inside the container to port **5050** on the host machine.  
   - This allows you to access the registry via [http://localhost:5050](http://localhost:5050) instead of the default `5000`.

4. **`--restart unless-stopped`**  
   - Ensures the container **restarts automatically** unless it is explicitly stopped.  
   - If the system reboots, the container will start again unless stopped manually.

5. **`--name local-docker-registry`**  
   - Assigns the name **`local-docker-registry`** to the container, making it easier to manage.

6. **`-v /mnt/sdb2-partition/dockerImages:/var/lib/registry`**  
   - **`-v`** mounts a volume (bind mount).  
   - Maps the host directory **`/mnt/sdb2-partition/dockerImages`** to **`/var/lib/registry`** inside the container.  
   - This ensures that images pushed to the registry are **stored persistently** on the host.

7. **`-e REGISTRY_STORAGE_DELETE_ENABLED=true`**  
   - **`-e`** sets an environment variable inside the container.  
   - `REGISTRY_STORAGE_DELETE_ENABLED=true` enables the **deletion of images** from the registry.

8. **`registry:2`**  
   - Specifies the **Docker image** to use (`registry:2` is the official **Docker Registry version 2**).
  
## Before Pushing Images

When using images from a **local registry**, there are a few additional considerations to ensure smooth integration with Docker and Kubernetes.

### 1. Insecure Registry Configuration

If your local registry does not use TLS (i.e., it's running over HTTP), you'll need to configure Docker (and Kubernetes nodes if using Docker as the runtime) to trust it as an insecure registry.

### Docker Configuration:
- **Edit Docker Daemon Configuration**
  Update or create `/etc/docker/daemon.json` on your host (or each Kubernetes node) with the following:
  ```json
  {
    "insecure-registries" : ["<registry-host>:<port>"]
  }
  ```
  
  Replace <registry-host>:<port> (e.g., localhost:5050 or the actual IP address if accessed remotely).
  Example:
  ```json
  {
  "log-driver": "syslog",
  "insecure-registries": ["192.168.1.10:5050","192.168.1.110:5050"]
  }
  ```
  
  Restart Docker
  ```sh
  sudo systemctl restart docker
  ```

### 2. (Alternative to above insecure) Secure Registry Configuration

- If you don’t have a certificate from a trusted CA, you can generate a **self-signed certificate** for your registry.
  You need an SSL certificate to enable HTTPS. You can either get one from a trusted CA (Let's Encrypt, etc.) or generate a self-signed certificate.

   ```sh
   mkdir -p /mnt/sdb2-partition/certs
   
   openssl req -newkey rsa:4096 -nodes -sha256 -keyout /mnt/sdb2-partition/certs/domain.key \
       -x509 -days 365 -out /mnt/sdb2-partition/certs/domain.crt \
       -subj "/CN=$(hostname -I | awk '{print $1}')"
   ```
   
   This creates:
   
   domain.key → Private key
   domain.crt → Self-signed certificate valid for 1 year
   (Make sure to replace /mnt/sdb2-partition/certs with your preferred location.)

- Run the Secure Docker Registry
  Now, restart the registry with TLS enabled, using the same stored images directory.

  ```sh
  docker run -d -p 5050:5000 --restart unless-stopped --name local-docker-registry \
  -v /mnt/sdb2-partition/dockerImages:/var/lib/registry \
  -v /mnt/sdb2-partition/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  registry:2
  ```
   This ensures:
   
   The existing images are retained (/mnt/sdb2-partition/dockerImages remains unchanged).
   The registry now runs with HTTPS instead of HTTP.
#
#
#
#
#
#
#




### 2. Further Changes Before Pushing Image

When preparing to push images to a **local Docker registry**, you need to ensure that your setup is correctly configured and that your images are properly tagged. Here are some important steps:

1. **`Tag Your Image Correctly`**

Before pushing, make sure your image is tagged with the registry’s address and port. For example:

```sh
docker tag my-image:latest <registry-host>:<port>/my-image:latest
```
Replace <registry-host>:<port> (e.g., localhost:5050 or an IP address) with the appropriate values.

2. **`Authentication Considerations`**

If your local registry requires authentication, ensure that you:

- **Log in to the registry:**
  ```sh
  docker login <registry-host>:<port>
  ```
- **`Provide the correct username and password when prompted.`**


### 3. Verify Network Accessibility

- **For Docker Clients:**
  Ensure that the registry is accessible over the network at the specified host and port. This is crucial especially if you're pushing from a different machine.

- **DNS and Firewall Settings:**
  Check that DNS resolution or IP addressing is configured correctly and that firewall settings permit traffic on the registry port.


### 4. Confirm Persistent Storage and Configuration

- **Persistent Data:**
Ensure that the registry container is set up with a volume mount (e.g., -v /mnt/sdb2-partition/dockerImages:/var/lib/registry) so that your images are stored persistently.

- **Environment Variables:**
If you require additional features like enabling image deletion (-e REGISTRY_STORAGE_DELETE_ENABLED=true), confirm that these are configured correctly in the registry container.


### 5. Ensure Network Accessibility

- ***For Docker:***
  Ensure the registry is reachable at the specified host and port.

- ***For Kubernetes:***
  - If the registry is running on your local machine, ensure the nodes can access it using the correct IP address or DNS name.
  - Update your image references accordingly if localhost is not applicable in a multi-node environment.
 

### 6. Image Pull Secrets (If Using Authentication)***

If you have enabled authentication on your private registry, you need to create a Docker registry secret in Kubernetes to allow pulling images.

- **Create an Image Pull Secret**
  
   ```sh
   kubectl create secret docker-registry my-registry-secret \
     --docker-server=<registry-host>:<port> \
     --docker-username=<username> \
     --docker-password=<password> \
     --docker-email=<email>
   ```
   Replace <registry-host>:<port>, <username>, <password>, and <email> with your actual registry details.
  
- **Reference the Secret in Your Pod Specs***
  
  In your pod or deployment YAML, add the secret under imagePullSecrets

  ```yaml
  spec:
  containers:
    - name: my-container
      image: <registry-host>:<port>/my-image:tag
  imagePullSecrets:
    - name: my-registry-secret
  ```





