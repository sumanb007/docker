# Docker Setups

## Table of Contents
1. [Docker Private Repository](#docker-private-repository)
2. [Before Pushing Images](#before-pushing-images)
3. [Dockerfile and Multi Stage Build](#dockerfile-and-multi-stage-build)

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

### 1. Alternative-1: Insecure Registry Configuration

If your local registry does not use TLS (i.e., it's running over HTTP), you'll need to configure Docker (and Kubernetes nodes if using Docker as the runtime) to trust it as an insecure registry.

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

### 2. Alternative-2: Secure Registry Configuration

- If you don’t have a certificate from a trusted CA, you can generate a **self-signed certificate** for your registry.
  You need an SSL certificate to enable HTTPS. You can either get one from a trusted CA (Let's Encrypt, etc.) or generate a self-signed certificate.

   ```sh
   mkdir -p /mnt/sdb2-partition/certs
   
   openssl req -newkey rsa:4096 -nodes -sha256 -keyout /mnt/sdb2-partition/certs/domain.key \
       -x509 -days 365 -out /mnt/sdb2-partition/certs/domain.crt \
       -subj "/CN=$(hostname -I | awk '{print $1}')"
   ```
   
   This creates:
   
   - domain.key → Private key
   - domain.crt → Self-signed certificate valid for 1 year (you may change '-days 1825' for 5 years)

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
  - The existing images are retained (/mnt/sdb2-partition/dockerImages remains unchanged).
  - The registry now runs with HTTPS instead of HTTP.
    

### 3. Trust the Registry Certificate (On Clients)

- For Self-Signed Certificates (On Linux)

  In our case, since we switched to HTTPS, clients (your local machine, Kubernetes nodes, etc.) must trust the new certificate.

  ```sh
  mkdir -p /etc/docker/certs.d/$(hostname -I | awk '{print $1}')
  cp /mnt/sdb2-partition/certs/domain.crt /etc/docker/certs.d/$(hostname -I | awk '{print $1}')/ca.crt
  ```

  Then, restart Docker:
  ```
  sudo systemctl restart docker
  ```

  If other machines are accessing the registry, copy domain.crt to each client and install it under /etc/docker/certs.d/<registry-ip>/ca.crt.

  Since our Docker Registry is hosted on 192.168.1.110, and other hosts (192.168.1.11, 192.168.1.12, 192.168.1.13) need to access it securely.
  Follow these steps to distribute and trust the self-signed certificate on each client machine.

  On the main registry host (192.168.1.110), run:

  ```sh
  scp /mnt/sdb2-partition/certs/domain.crt user@192.168.1.11:/tmp/
  scp /mnt/sdb2-partition/certs/domain.crt user@192.168.1.12:/tmp/
  scp /mnt/sdb2-partition/certs/domain.crt user@192.168.1.13:/tmp/
  ```
  (Replace user with your actual username on the target machines.)

  SSH into each client (192.168.1.11, .12, .13) and move the certificate to the correct location:

  ```sh
  sudo mkdir -p /etc/docker/certs.d/192.168.1.110
  sudo mv /tmp/domain.crt /etc/docker/certs.d/192.168.1.110/ca.crt
  sudo systemctl restart docker
  ```

- Test the Connection

  - To verify that the registry is working, list available repositories before pulling an image.

    ```sh
    curl -k https://192.168.1.110:5050/v2/_catalog
    ```

    Expected Output
    ```sh
    {"repositories":["frontend-crud-webapp", "backend-crud-webapp"]}
    ```
    
  - To list Tags for a Specific Repository

    ```sh
    curl -k https://192.168.1.110:5050/v2/frontend-crud-webapp/tags/list
    ```

    Expected Output
    ```sh
    {"name":"frontend-crud-webapp","tags":["v1","v2","latest"]}
    ```

  - Check If a Specific Image Exists:

    ```sh
    curl -k -H "Accept: application/vnd.docker.distribution.manifest.v2+json" -I -X GET https://192.168.1.110:5050/v2/backend-crud-webapp/manifests/v1
    ```

    If the image exists, it will show as below.
    ```sh
    HTTP/2 200
    content-type: application/vnd.docker.distribution.manifest.v1+prettyjws
    docker-content-digest: sha256:919d949c1500219df0f03d6203b5f6ca896a11cb4b9bfa2cca3a8c2e775dccb5
    docker-distribution-api-version: registry/2.0
    etag: "sha256:919d949c1500219df0f03d6203b5f6ca896a11cb4b9bfa2cca3a8c2e775dccb5"
    x-content-type-options: nosniff
    content-length: 15448
    date: Wed, 12 Mar 2025 11:35:46 GMT
    ```

    If not,
    ```sh
    HTTP/2 404
    ```

    You can further verify by pulling and pushing new images.


### 4. Further Changes Before Pushing Image

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


### 5. Verify Network Accessibility

- **For Docker Clients:**
  Ensure that the registry is accessible over the network at the specified host and port. This is crucial especially if you're pushing from a different machine.

- **DNS and Firewall Settings:**
  Check that DNS resolution or IP addressing is configured correctly and that firewall settings permit traffic on the registry port.


### 6. Confirm Persistent Storage and Configuration

- **Persistent Data:**
Ensure that the registry container is set up with a volume mount (e.g., -v /mnt/sdb2-partition/dockerImages:/var/lib/registry) so that your images are stored persistently.

- **Environment Variables:**
If you require additional features like enabling image deletion (-e REGISTRY_STORAGE_DELETE_ENABLED=true), confirm that these are configured correctly in the registry container.


### 7. Ensure Network Accessibility

- ***For Docker:***
  Ensure the registry is reachable at the specified host and port.

- ***For Kubernetes:***
  - If the registry is running on your local machine, ensure the nodes can access it using the correct IP address or DNS name.
  - Update your image references accordingly if localhost is not applicable in a multi-node environment.
 

### 8. Image Pull Secrets (If Using Authentication)***

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

## Dockerfile and Multi Stage Build

A Docker multi-stage build is a technique used to create smaller and more efficient Docker images by using multiple FROM statements in a Dockerfile. This approach allows us to build an application in one stage and copy only the necessary artifacts into a final, minimal image.

### Why Use Multi-Stage Builds?
- Smaller Image Size 🏋️‍♂️ → Reduces unnecessary dependencies in the final image.
- Improved Security 🔒 → Minimizes attack surface by removing build tools.
- Better Performance 🚀 → Less bloat means faster deployment and startup times.

> [!TIP]
> When to Use Multi-Stage Builds?
> - When your application requires compilation (Go, C, C++, Java, Rust).
> - When using package managers (npm, pip, maven) and you don’t need dev (or build tools) dependencies in production.
> - To leverage(control) Docker layer caching for faster rebuilds.
> - When you want to create small, optimized images for faster deployment.
> - Security: Fewer components reduce attack surfaces.
> - Optimize Dockerfile and keep them easy to read and maintain.

### Let's Continue

Now we have three tier web application, frontend, backend and mongodb.
We are are converting the traditional Dockerfile into optimized final image using multi-stage build.

Here, we have one Dockerfile for frontend.
This is not ideal for long-term projects, but it works if you just need to install specific packages without setting up package.json.
The image created will have ~1.5G size.
```dockerfile
FROM 	node:latest
WORKDIR	/app
COPY	. .
RUN	npm i react-bootstrap@next bootstrap@5.1.0 react-router-dom axios formik yup
EXPOSE	3000
CMD	npm start
```

Alternatively, we have another ideal Dockerfile. This will create ~434MB size.
```dockerfile
FROM node:latest
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

For the frontend, a common practice is to build the application into static files and serve them with a lightweight web server (e.g., Nginx).
Now lets convert alternative Dockerfile into multi-stage build. This will create ~54MB size.
```dockerfile
# Stage 1: Build the React app
FROM node:16-alpine AS builder
WORKDIR /frontend
COPY package*.json ./
RUN npm ci --silent
COPY . .
RUN npm run build  # Assumes "build" script exists in package.json

# Stage 2: Serve with NGINX
FROM nginx:alpine
#Copies only the built files from the first stage to Nginx's HTML directory.
COPY --from=builder /frontend/build /usr/share/nginx/html
# Custom NGINX config for port 8080 and React routing
COPY nginx-frontend.conf /etc/nginx/conf.d/default.conf
# Explicitly expose port 8080 (optional but good practice)
EXPOSE 8080
```
- node:alpine → A minimal version of Node.js (~40MB), reducing image size.
- AS builder → Creates a named build stage for a multi-stage build.
- COPY package*.json ./ → Only copies package files first, helping Docker cache dependencies.
- npm ci --silent → Installs dependencies cleanly (ensures exact versions). silent: Suppresses unnecessary output, making logs cleaner.
- COPY . . → Copies the rest of the application files.
- npm run build → Creates the production-ready static files in /app/build.

Why This is Better
- Uses lighter node:alpine.
- Only production dependencies are installed, keeping it minimal.
- No local node_modules copied, avoiding conflicts.
- No npm start → Instead, directly starts with node server.js (faster & efficient).

### Similarly for Backend

```dockerfile
# Stage 1: Install dependencies
FROM node:16-alpine AS dependencies
WORKDIR /backend
COPY package*.json ./
RUN npm ci --only=production --silent

# Stage 2: Copy only production files
FROM node:16-alpine
WORKDIR /backend
COPY --from=dependencies /backend/node_modules ./node_modules
COPY . .
CMD ["npm", "start"]
```





