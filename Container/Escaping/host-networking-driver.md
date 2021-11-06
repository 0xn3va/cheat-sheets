If a container was configured with the Docker [host networking driver (--network=host)](https://docs.docker.com/network/host/), that container's network stack is not isolated from the Docker host (the container shares the host's networking namespace), and the container does not get its own IP-address allocated. In other words, the container binds all services directly to the host's IP. Furthermore the container can intercept ALL network traffic that the host is sending and receiving on shared interface `tcpdump -i eth0`.

For instance, you can use this to sniff and even spoof traffic between host and metadata instance.

References:

- [Writeup: How to contact Google SRE: Dropping a shell in cloud SQL](https://offensi.com/2020/08/18/how-to-contact-google-sre-dropping-a-shell-in-cloud-sql/)
- [Metadata service MITM allows root privilege escalation (EKS / GKE)](https://blog.champtar.fr/Metadata_MITM_root_EKS_GKE/)
