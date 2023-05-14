# Overview

PID namespace provides separation between processes. It prevents system processes from being visible and allows process IDs to be reused including PID 1. If the host's PID namespace is shared with containers, it would allow these to see all of the processes on the host system.

If the PID namespace is shared it means that:

- The container process no longer has PID 1. Some containers refuse to start without PID 1 (for example, containers using `systemd`) or run commands like `kill -HUP 1` to signal the container process. In Kubernetes pods with a shared process namespace, `kill -HUP 1` will signal the pod sandbox.
- Host processes (or processes of containers in a pod) are visible to containers (or other containers). This includes all information visible in `/proc`, such as secrets that were passed as arguments or environment variables. These are protected only by regular Unix permissions.
- Host (or container) filesystems are visible to containers (or other containers in the pod) through the `/proc/[pid]/root` link.

PID namespace sharing decreases process level isolation between the host and the containers and makes possible various kinds of attacks including injection into a host's process (requires [CAP_SYS_PTRACE](/Container/Escaping/excessive-capabilities.md#cap_sys_ptrace)).

# References

- [tbhaxor: Container Breakout – Part 1 (LAB: Process Injection)](https://tbhaxor.com/container-breakout-part-1/)
- [#BrokenSesame: Accidental ‘write’ permissions to private registry allowed potential RCE to Alibaba Cloud Database Services](https://www.wiz.io/blog/brokensesame-accidental-write-permissions-to-private-registry-allowed-potential-r)
