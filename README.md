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

