# Docker Setups

## Table of Contents
1. [Docker Private Repository](#docker-private-repository)
2. [Multi Stage Build](#multi-stage-build)

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


