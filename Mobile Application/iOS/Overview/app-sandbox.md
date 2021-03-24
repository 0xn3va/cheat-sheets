On iOS, all third-party applications are "sandboxed", so they are restricted from accessing files stored by other applications or from making changes to the device. Sandboxing prevents applications from gathering or modifying information stored by other applications. Each application has a unique home directory for its files, which is randomly assigned when the application is installed. If a third-party application needs to access information other than its own, it does so only by using services explicitly provided by iOS.

System files and resources are also shielded from the user's applications. The majority of iOS run as the non-privileged user `mobile`, as do all third-party applications. The entire OS partition is mounted as read-only. Unnecessary tools, such as remote login services, are not included in the system software, and APIs do not allow applications to escalate their own privileges to modify other applications or iOS.

# Protections

The restrictions in an App's "jail" include, but are not limited to:
- Inability to break out of the application's directory. The application sees its own bundle container `/var/containers/Bundle/Application/<app-GUID>/` as the root, similar to the [chroot(2)](https://man7.org/linux/man-pages/man2/chroot.2.html) system call. As a result, the application has no knowledge of any other installed applications, and cannot access system files.
- Inability to access any other process on the system, even if that process is owned by the same UID. The application sees itself as the only process executing on the system.
- Inability to directly use any of the hardware devices (camera, GPS, and etc.) without going through Apple's Frameworks (which in turn can impose restrictions).
- Inability to dynamically generate code. The low-level implementations of the [mmap(2)](https://man7.org/linux/man-pages/man2/mmap.2.html) and [mprotect(2)](https://man7.org/linux/man-pages/man2/mprotect.2.html) system calls (Mach's `vm_map_enter` and `vm_map_protect`, respectively) are intentionally modified to prevent any attempts to make writable memory pages executable as well. Combined with code signing and FairPlay, this imposes strong restrictions on what code can be run.
- Inability to perform any operations. For the `mobile` user, only certain operations are allowed. Root permissions for an applications (other than Apple's own) are not possible.

# References

- [Apple Platform Security: Sandboxing](https://support.apple.com/en-gb/guide/security/sec15bfe098e/web)
- [Mac OS X and iOS Internals: To the Apple's Core by Jonathan Levin](https://www.amazon.com/gp/product/1118057651/)
