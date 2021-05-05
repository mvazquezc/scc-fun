# **AllowPrivilegeEscalation**

In this demo we are going to show how privilege escalation can be managed through the use of _no_new_privs_ in order to avoid use of SUID binaries and binaries with file capabilities.

## **Hands-on Demo 1**

In this demo we will see how _no_new_privs_ can be used to avoid SUID binaries.

1. Create a small container image that has the `whoami` binary configured with SETUID bit

    ~~~sh
    cat <<EOF > whoami-setuid.dockerfile
    FROM fedora:32
    RUN chmod +s /usr/bin/whoami
    ENTRYPOINT /usr/bin/whoami
    EOF
    ~~~

    ~~~sh
    podman build -f whoami-setuid.dockerfile -t whoami-setuid
    ~~~
2. If we run the image as user `1024` without setting the _no_new_privs_ bit this is what we get

    ~~~sh
    podman run -it --rm --user=1024 whoami-setuid
    ~~~

    > **NOTE**: As you can see, the privilege escalation happened.

    ~~~
    root
    ~~~
3. If we run the image as user 1024 with the _no_new_privs_ bit set

    ~~~sh
    podman run -it --rm --user=1024 --security-opt=no-new-privileges whoami-setuid
    ~~~

    > **NOTE**: In this case, the privilege escalation was blocked.

    ~~~
    1024
    ~~~

## **Hands-on Demo 2**

In this demo we will see how _no_new_privs_ can be used to avoid users to use binaries with file capabilities if those capabilities are not in their `permitted` and `effective` thread's capability set.

1. Run the container with user 1024 and without setting the no_new_privs bit

    > **NOTE**: The image we are using for our test has a small web service with the `NET_BIND_SERVICE` file capability configured in the binary.

    ~~~sh
    podman run --rm -it --entrypoint /bin/bash --user 1024 -e APP_PORT=80 --name reversewords-test quay.io/mavazque/reversewords-captest:latest
    ~~~
2. Get the file capabilities for the web service binary

    ~~~sh
    getcap /usr/bin/reverse-words
    ~~~

    ~~~
    /usr/bin/reverse-words = cap_net_bind_service+ep
    ~~~
3. Get the container's thread capabilities

    ~~~sh
    grep Cap /proc/1/status
    ~~~

    ~~~
    CapInh: 00000000800405fb
    CapPrm: 0000000000000000
    CapEff: 0000000000000000
    CapBnd: 00000000800405fb
    CapAmb: 0000000000000000
    ~~~
4. Decode thread capabilities

    ~~~sh
    capsh --decode=00000000800405fb
    ~~~

    ~~~
    0x00000000800405fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_setfcap
    ~~~
5. Execute the binary

    ~~~sh
    /usr/bin/reverse-words
    ~~~

    > **NOTE**: As expected, the binary was able to get the `NET_BIND_SERVICE` capability

    ~~~
    2021/04/23 19:30:58 Starting Reverse Api v0.0.18 Release: NotSet
    2021/04/23 19:30:58 Listening on port 80
    ~~~
6. We are going to run the container with the _no_new_privs` bit set

    ~~~sh
    podman run --rm -it --entrypoint /bin/bash --security-opt no-new-privileges --user 1024 -e APP_PORT=80 --name reversewords-test quay.io/mavazque/reversewords-captest:latest
    ~~~
7. File caps and thread caps remain the same as in the previous run, let's run the binary

    ~~~sh
    /usr/bin/reverse-words
    ~~~

    > **NOTE**: This time the binary couldn't get the capability into the thread's effective set due to the _no_new_privs_ bit

    ~~~
    2021/04/23 19:31:42 Starting Reverse Api v0.0.18 Release: NotSet
    2021/04/23 19:31:42 Listening on port 80
    2021/04/23 19:31:42 listen tcp :80: bind: permission denied
    ~~~
8. If we run with root this time and with no_new_privs

    ~~~sh
    podman run --rm -it --entrypoint /bin/bash --security-opt no-new-privileges --user 0 -e APP_PORT=80 --name reversewords-test quay.io/mavazque/reversewords-captest:latest
    ~~~
9. Get the container's thread capabilities

    ~~~sh
    grep Cap /proc/1/status
    ~~~

    ~~~
    CapInh: 00000000800405fb
    CapPrm: 00000000800405fb
    CapEff: 00000000800405fb
    CapBnd: 00000000800405fb
    CapAmb: 0000000000000000
    ~~~
10. Decode thread capabilities

    ~~~sh
    capsh --decode=00000000800405fb
    ~~~

    ~~~
    0x00000000800405fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_setfcap
    ~~~
11. This time we got the NET_BIND_SERVICE in the thread's effective set, which means we will be able to use it since we don't need to raise it

    ~~~sh
    /usr/bin/reverse-words
    ~~~

    ~~~
    2021/04/23 19:33:05 Starting Reverse Api v0.0.18 Release: NotSet
    2021/04/23 19:33:05 Listening on port 80
    ~~~
