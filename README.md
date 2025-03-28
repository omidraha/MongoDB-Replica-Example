# MongoDB Replica Set Setup on Three Nodes Using Docker

This guide explains how to deploy a MongoDB replica set on three separate nodes running Docker. Each node will run a MongoDB container that mounts a local directory (hostPath) for data persistence. Although hostPath is not the ideal solution for production (network-attached storage is preferred), this guide demonstrates how to configure your environment when no other storage is available.

```
                      +------------------+
                      |   Replica Set:   |
                      |      rs0         |
                      +------------------+

                            (Primary)
                         +-------------+
                         |             |
                         |  Node 1     |
                         |  192.168.1.1|
                         |  MongoDB    |
                         |  Container  |
                         +-------------+
                               ▲
                               |
             +----------------+----------------+
             |                                 |
             |                                 |
        (Secondary)                      (Secondary)
     +-------------+                   +-------------+
     |             |                   |             |
     |  Node 2     |                   |  Node 3     |
     | 192.168.1.2 |                   | 192.168.1.3 |
     |  MongoDB    |                   |  MongoDB    |
     |  Container  |                   |  Container  |
     +-------------+                   +-------------+

         All nodes:
       - Run in Docker
       - Use `/data/db` as host volume
       - Expose port 27017
       - Belong to replica set "rs0"
```

---

## Prerequisites

- **Three nodes** with IP addresses:  
  - Node 1: `192.168.1.1`  
  - Node 2: `192.168.1.2`  
  - Node 3: `192.168.1.3`
- Docker installed on each node.
- Each node has a local directory (e.g., `/data/db`) for MongoDB data.
- Ensure network connectivity between nodes on port `27017`.

---

## Step 1. Prepare the Nodes

On each node, create and set the permissions for the data directory:

```bash
sudo mkdir -p /data/db
sudo chown -R $USER:$USER /data/db
```

---

## Step 2. Run MongoDB Container on Each Node

On **each node**, run the following command to start a MongoDB container configured for a replica set:

```bash
docker run -d \
  --name mongo \
  -v /data/db:/data/db \
  -p 27017:27017 \
  mongo:7.0 \
  --replSet rs0 --bind_ip_all
```

- **Explanation:**
  - `--name mongo`: Names the container (change if running multiple containers on the same node).
  - `-v /data/db:/data/db`: Mounts the host directory `/data/db` into the container.
  - `-p 27017:27017`: Exposes port 27017 on the host.
  - `mongo:7.0`: Uses MongoDB version 7.0 image.
  - `--replSet rs0`: Configures the container as part of replica set `rs0`.
  - `--bind_ip_all`: Listens on all network interfaces so that nodes can communicate.

Repeat this step on Node 1, Node 2, and Node 3.

---

## Step 3. Initiate the Replica Set

1. **Choose one node (e.g., Node 1) to initiate the replica set.**  
   Connect to the MongoDB shell in the container on Node 1:

   ```bash
   docker exec -it mongo mongosh --host 127.0.0.1:27017
   ```

2. **Create the replica set configuration document in the shell:**

   ```javascript
   cfg = {
     _id: "rs0",
     members: [
       { _id: 0, host: "192.168.1.1:27017" },
       { _id: 1, host: "192.168.1.2:27017" },
       { _id: 2, host: "192.168.1.3:27017" }
     ]
   }
   ```

3. **Initiate the replica set:**

   ```javascript
   rs.initiate(cfg)
   ```

4. **Verify the replica set status:**

   ```javascript
   rs.status()
   ```

   You should see one node as **PRIMARY** and the others as **SECONDARY**.

---

## Step 4. Configure Client Connections

When connecting your application (for example, a Node.js app), use a connection string that includes all three nodes and specifies the replica set name:

```
mongodb://192.168.1.1:27017,192.168.1.2:27017,192.168.1.3:27017/?replicaSet=rs0
```

The MongoDB driver will automatically discover the current primary and route write operations accordingly. If the primary fails, the driver will detect a new primary once the replica set elects one.

---

## Step 5. Production Considerations

With three data-bearing nodes, the replica set can form a majority even if one node fails.


---

## Conclusion

This README guides you through setting up a MongoDB replica set on three separate nodes using Docker containers with local hostPath storage. By following these steps, you ensure that your MongoDB deployment will:
- Have data redundancy and high availability.
- Automatically elect a new primary in the event of a node failure.
- Use Docker for deployment while relying on local storage per node.

