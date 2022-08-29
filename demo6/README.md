# **AllowPrivilegeEscalation**

In this demo we are going to show how privilege escalation can be managed through the use of _no_new_privs_ in order to avoid use of SUID binaries and binaries with file capabilities.

## **Hands-on Demo 1**

> **NOTE**: Below tests were run in a Fedora 35 machine with Podman v3.4.4, results may vary when using other O.S / Podman version.

In this demo we will see how _no_new_privs_ can be used to avoid SUID binaries.

1. Create a small container image that has the `whoami` binary configured with SETUID bit

    ~~~sh
    cat <<EOF > whoami-setuid.dockerfile
    FROM fedora:36
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

    ~~~sh
    root
    ~~~

3. If we run the image as user 1024 with the _no_new_privs_ bit set

    ~~~sh
    podman run -it --rm --user=1024 --security-opt=no-new-privileges whoami-setuid
    ~~~

    > **NOTE**: In this case, the privilege escalation was blocked.

    ~~~sh
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

    ~~~sh
    /usr/bin/reverse-words = cap_net_bind_service+ep
    ~~~

3. Get the container's thread capabilities

    ~~~sh
    grep Cap /proc/1/status
    ~~~

    ~~~sh
    CapInh: 0000000000000000
    CapPrm: 0000000000000000
    CapEff: 0000000000000000
    CapBnd: 00000000800405fb
    CapAmb: 0000000000000000
    ~~~

4. Decode thread capabilities

    ~~~sh
    capsh --decode=00000000800405fb
    ~~~

    ~~~sh
    0x00000000800405fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_setfcap
    ~~~

5. Execute the binary

    ~~~sh
    /usr/bin/reverse-words
    ~~~

    > **NOTE**: As expected, the binary was able to get the `NET_BIND_SERVICE` capability

    ~~~sh
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

    > **NOTE**: This time the binary couldn't get the capability into the thread's effective set due to the _no_new_privs_ bit.e

    ~~~sh
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

    ~~~sh
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

    ~~~sh
    0x00000000800405fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_setfcap
    ~~~

11. This time we got the NET_BIND_SERVICE in the thread's effective set, which means we will be able to use it since we don't need to raise it

    ~~~sh
    /usr/bin/reverse-words
    ~~~

    ~~~sh
    2021/04/23 19:33:05 Starting Reverse Api v0.0.18 Release: NotSet
    2021/04/23 19:33:05 Listening on port 80
    ~~~

## **Hands-on Demo 3**

In this demo we're going to show how we can audit containers abusing setuid/sudo on an OpenShift cluster.

Even if file capabilities can cause a privilege escalation we have great tools today to avoid pods from getting such capabilities via SCCs, where the restricted-v2 SCC does a pretty good job limiting the number of CAPS available for pods by default. On the other hand, we have the "becoming root/executing as root" problem, which in this case new v2 SCCs introduced in OCP 4.11 are helping to mitigate, since them have the setting allowPrivilegeEscalation set to false.

In order to showcase a scenario where an application is abusing the setuid, kudos to [r/linuxquestions](https://www.reddit.com/r/linuxquestions/) for providing the testing app (quay.io/fherrman/my-ubi-setuid:0.4):

~~~C
#include <stdlib.h>
#include <unistd.h>

int main () {
    setuid(0);
    execl("/bin/bash", "bash", "-p",  0);
}
~~~

As you can see the container image will have this binary with setuid configured under /usr/bin/bashwrap:

~~~sh
$ ls -l /usr/bin/bashwrap 
-rwsr-xr-x. 1 root root 17488 Nov 12 22:42 /usr/bin/bashwrap
~~~

Now that we introduced the app that we will be using we will see how a user can build a container image with an app like the one we have above in order to become root with an SCC that doesn't allow running as root, but does allow privilege escalation.

1. Create a namespace for our tests

    ~~~sh
    oc create ns test-priv-esc
    ~~~

2. Deploy our application

    1. Since v2 SCCs restrict the privilege escalation we are going to provide access to the old restricted SCC to showcase this issue to the default SA in this project

        ~~~sh
        oc -n test-priv-esc adm policy add-scc-to-user restricted -z default
        ~~~

    2. Deploy the application

        ~~~sh
        cat <<EOF | oc -n test-priv-esc create -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          creationTimestamp: null
          labels:
            app: privescalation
          name: privescalation
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: privescalation
          strategy: {}
          template:
            metadata:
              creationTimestamp: null
              labels:
                app: privescalation
            spec:
              containers:
              - image: quay.io/fherrman/my-ubi-setuid:0.4
                command: ["sleep","9999"]
                name: my-ubi-setuid
                resources: {}
                securityContext:
                  allowPrivilegeEscalation: true
        status: {}
        EOF
        ~~~

3. If we check the SCC assigned to our pod we will see that we got the `restricted` SCC assigned to our workload:

    ~~~sh
    oc -n test-priv-esc get pod -l app=privescalation -o yaml | grep scc
    ~~~

    ~~~sh
    openshift.io/scc: restricted
    ~~~

4. The restricted SCC does not allow containers to run with a root uid (0), let's see how we configured our setuid binary and let's try to execute it:

    1. Connect to the pod:

        ~~~sh
        oc -n test-priv-esc rsh deployment/privescalation

        sh-4.4$
        ~~~

    2. Check app binary configuration:

        ~~~sh
        ls -l /usr/bin/bashwrap

        -rwsr-xr-x. 1 root root 17488 Nov 12 22:42 /usr/bin/bashwrap
        ~~~

    3. Execute the app:

        ~~~sh
        /usr/bin/bashwrap

        bash-4.4#
        ~~~

    4. Check our effective uid:

        > **NOTE**: As you can see our effective uid is 0.

        ~~~sh
        id
        
        uid=1000680000(1000680000) gid=0(root) euid=0(root) groups=0(root),1000680000
        ~~~

5. This means that at this point the container process is running as root in the node. Let's verify it:

    1. Check the node where the pod is running

        ~~~sh
        OCP_NODE=$(oc -n test-priv-esc get pod -l app=privescalation -o jsonpath='{.items[*].spec.nodeName}')
        ~~~

    2. Open a debug session into that node

        ~~~sh
        oc debug node/${OCP_NODE}
        chroot /host
        ~~~

    3. Connect to the pod in a different terminal, execute our app and run a "sleep 288":

        ~~~sh
        oc -n test-priv-esc rsh deployment/privescalation
        /usr/bin/bashwrap
        sleep 288
        ~~~

    4. Back in the node shell, check the process owner:

        ~~~sh
        ps -ef | grep "sleep 288" | grep -v grep
        ~~~

        > **NOTE**: As you can see we're running as root.

        ~~~sh
        root     4162199 4162111  0 16:48 pts/0    00:00:00 /usr/bin/coreutils --coreutils-prog-shebang=sleep /usr/bin/sleep 288
        ~~~

At this point we have demonstrated how we can abuse setuid binaries, now let's see what we can do on the node to know when this happens. 

1. We will use auditd rules to monitor when a process performs privilege escalation by changing from non-root uid to root uid. The audit rule we will use is the following one:

    > **NOTE**: Below rule only targets 64bits syscall, if you want to get the 32bits ones as well you need to edit the rule:

    ~~~sh
    always,exit -F arch=b64 -S execve -C uid!=euid -F euid=0 -k setuid-abuse
    ~~~

2. In order to get this rule to our worker nodes we will create the following MachineConfig:

    ~~~sh
    cat <<EOF | oc create -f -
    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfig
    metadata:
      labels:
        machineconfiguration.openshift.io/role: worker
      name: 99-auditd-setuid-rule
    spec:
      config:
        ignition:
          version: 3.1.0
        storage:
          files:
          - contents:
              source: data:text/plain;charset=utf-8;base64,LWEgYWx3YXlzLGV4aXQgLUYgYXJjaD1iNjQgLVMgZXhlY3ZlIC1DIHVpZCE9ZXVpZCAtRiBldWlkPTAgLWsgc2V0dWlkLWFidXNlCg==
            mode: 420
            overwrite: true
            path: /etc/audit/rules.d/setuid-abuse.rules
    EOF
    ~~~

3. After our nodes restart we will have the audit rules in place, if we run the same operation in our application pod, we will get something like this in our audit log:

    > **NOTE**: Below command should be run in the node (you can use oc debug node as we did before)

    ~~~sh
    grep setuid-abuse /var/log/audit/audit.log

    type=SYSCALL msg=audit(1637601395.568:124): arch=c000003e syscall=59 success=yes exit=0 a0=4006b0 a1=7ffee2255690 a2=7ffee2255818 a3=1 items=2 ppid=25611 pid=28357 auid=4294967295 uid=1000680000 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=4294967295 comm="bash" exe="/usr/bin/bash" subj=system_u:system_r:container_t:s0:c15,c26 key="setuid-abuse"ARCH=x86_64 SYSCALL=execve AUID="unset" UID="unknown(1000680000)" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
    ~~~

4. In the audit log we have different information that we can use to identify the container running the privilege escalation.

5. We could send this audit log to a SIEM system and create alerts based on this audit rules.