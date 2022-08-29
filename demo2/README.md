# **Secccomp Security Profiles**

In this demo we will learn how we can work with seccomp profiles for our containers.

We will start by generating a seccomp profile for a local container with podman, and then we will see how we can use that profile for pods running on OpenShift.

## **Creating a Seccomp Profile with Podman**

In order to generate our `seccomp` profile we are going to rely on an OCI Hook which will intercept the syscalls made by our container in order to get them added to the seccomp profile.

The OCI Hook project can be found here: https://github.com/containers/oci-seccomp-bpf-hook

1. Create a VM with CentOS 8 Streams to run our tests. I'm using [kcli](https://github.com/karmab/kcli).

    ~~~sh
    kcli create vm -i centos8stream -P numcpus=2 -P memory=4096
    ~~~

2. Connect to the VM and install `podman` and the `oci-seccomp-bpf-hook`

    > **NOTE**: We need to install kernel-core to get the `kheaders` module required by the hook.

    ~~~sh
    # Install required tools
    sudo dnf install -y podman oci-seccomp-bpf-hook jq kernel-core kernel-headers
    # Reboot the VM
    sudo reboot
    ~~~

3. After reboot, load the `kheaders` module.

    ~~~sh
    sudo modprobe kheaders
    ~~~

4. Let's run a test container that runs a ls command

    ~~~sh
    sudo podman run --rm --runtime /usr/bin/runc --annotation io.containers.trace-syscall="of:/tmp/ls.json" fedora:36 ls / > /dev/null
    ~~~

5. The hook has generated the required `seccomp` profile at `/tmp/ls.json` let's review it.

    1. You can see the syscalls that ls needed in order to list the / filesystem

        ~~~sh
        cat /tmp/ls.json | jq '.syscalls[0].names'
        ~~~

6. Now let's try to use this `seccomp` profile to run the container

    ~~~sh
    sudo podman run --rm --runtime /usr/bin/runc --security-opt seccomp=/tmp/ls.json fedora:36 ls /
    ~~~

7. The container is now running which the minimum possible set of syscall allowed, let's try to change the ls command a bit:

    ~~~sh
    sudo podman run --rm --runtime /usr/bin/runc --security-opt seccomp=/tmp/ls.json fedora:36 ls -l /
    ~~~

    ~~~sh
    ls: /: Operation not permitted
    ~~~

    1. It seems that ls uses additional syscalls with the -l parameter. It's very important that you do an exhaustive testing of your application when tracing the syscalls required by it. Otherwise, you will find issues similar to this.

8. The hook allow us to pass an input file that will be used as baseline, then we will log the required additional syscalls into a new output file:

    ~~~sh
    sudo podman run --rm --runtime /usr/bin/runc --annotation io.containers.trace-syscall="if:/tmp/ls.json;of:/tmp/lsl.json" fedora:36 ls -l / > /dev/null
    ~~~

9. Now we should have a valid seccomp profile at `/tmp/lsl.json` that we can use for running our container:

    ~~~sh
    sudo podman run --rm --runtime /usr/bin/runc --security-opt seccomp=/tmp/lsl.json fedora:36 ls -l /
    ~~~

    ~~~sh
    total 8
    dr-xr-xr-x.   2 root root    6 Jan 20  2022 afs
    lrwxrwxrwx.   1 root root    7 Jan 20  2022 bin -> usr/bin
    dr-xr-xr-x.   2 root root    6 Jan 20  2022 boot
    drwxr-xr-x.   5 root root  340 Aug 25 17:15 dev
    drwxr-xr-x.   1 root root   25 Aug 25 17:15 etc
    drwxr-xr-x.   2 root root    6 Jan 20  2022 home
    lrwxrwxrwx.   1 root root    7 Jan 20  2022 lib -> usr/lib
    lrwxrwxrwx.   1 root root    9 Jan 20  2022 lib64 -> usr/lib64
    drwx------.   2 root root    6 Jul 19 07:48 lost+found
    drwxr-xr-x.   2 root root    6 Jan 20  2022 media
    drwxr-xr-x.   2 root root    6 Jan 20  2022 mnt
    ...
    ~~~

## **Using Seccomp Profiles on OpenShift**

At this point we have a valid seccomp profile, how can we get it added to OpenShift and start using it in our workloads? - Let's see.

### **Using the Seccomp profile in a workload**

1. First, we need to make sure that the seccomp profile exists at the default directory `/var/lib/kubelet/seccomp/`. We need to create a MachineConfig (MC) for that.

    > **NOTE:** You can use `cat /tmp/ls.json | jq . | base64 -w0` to encode the file to base64

    ~~~sh
    cat <<EOF | oc create -f -
    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfig
    metadata:
      labels:
        machineconfiguration.openshift.io/role: worker
      name: 99-seccomp-profile-ls
    spec:
      config:
        ignition:
          version: 3.1.0
        storage:
          files:
          - contents:
              source: data:text/plain;charset=utf-8;base64,ewogICJkZWZhdWx0QWN0aW9uIjogIlNDTVBfQUNUX0VSUk5PIiwKICAiYXJjaGl0ZWN0dXJlcyI6IFsKICAgICJTQ01QX0FSQ0hfWDg2XzY0IgogIF0sCiAgInN5c2NhbGxzIjogWwogICAgewogICAgICAibmFtZXMiOiBbCiAgICAgICAgImFjY2VzcyIsCiAgICAgICAgImFyY2hfcHJjdGwiLAogICAgICAgICJicmsiLAogICAgICAgICJjYXBnZXQiLAogICAgICAgICJjYXBzZXQiLAogICAgICAgICJjaGRpciIsCiAgICAgICAgImNsb3NlIiwKICAgICAgICAiZHVwMyIsCiAgICAgICAgImVwb2xsX2N0bCIsCiAgICAgICAgImVwb2xsX3B3YWl0IiwKICAgICAgICAiZXhlY3ZlIiwKICAgICAgICAiZXhpdF9ncm91cCIsCiAgICAgICAgImZjaGRpciIsCiAgICAgICAgImZjaG93biIsCiAgICAgICAgImZjbnRsIiwKICAgICAgICAiZnN0YXQiLAogICAgICAgICJmc3RhdGZzIiwKICAgICAgICAiZnV0ZXgiLAogICAgICAgICJnZXRkZW50czY0IiwKICAgICAgICAiZ2V0cGlkIiwKICAgICAgICAiZ2V0cHBpZCIsCiAgICAgICAgImdldHJhbmRvbSIsCiAgICAgICAgImlvY3RsIiwKICAgICAgICAibW1hcCIsCiAgICAgICAgIm1vdW50IiwKICAgICAgICAibXByb3RlY3QiLAogICAgICAgICJtdW5tYXAiLAogICAgICAgICJuYW5vc2xlZXAiLAogICAgICAgICJuZXdmc3RhdGF0IiwKICAgICAgICAib3BlbmF0IiwKICAgICAgICAicGlwZTIiLAogICAgICAgICJwaXZvdF9yb290IiwKICAgICAgICAicHJjdGwiLAogICAgICAgICJwcmVhZDY0IiwKICAgICAgICAicHJsaW1pdDY0IiwKICAgICAgICAicmVhZCIsCiAgICAgICAgInJzZXEiLAogICAgICAgICJydF9zaWdyZXR1cm4iLAogICAgICAgICJzZWNjb21wIiwKICAgICAgICAic2V0X3JvYnVzdF9saXN0IiwKICAgICAgICAic2V0X3RpZF9hZGRyZXNzIiwKICAgICAgICAic2V0Z2lkIiwKICAgICAgICAic2V0Z3JvdXBzIiwKICAgICAgICAic2V0aG9zdG5hbWUiLAogICAgICAgICJzZXR1aWQiLAogICAgICAgICJzdGF0ZnMiLAogICAgICAgICJzdGF0eCIsCiAgICAgICAgInRna2lsbCIsCiAgICAgICAgInVtYXNrIiwKICAgICAgICAidW1vdW50MiIsCiAgICAgICAgIndyaXRlIgogICAgICBdLAogICAgICAiYWN0aW9uIjogIlNDTVBfQUNUX0FMTE9XIiwKICAgICAgImFyZ3MiOiBbXSwKICAgICAgImNvbW1lbnQiOiAiIiwKICAgICAgImluY2x1ZGVzIjoge30sCiAgICAgICJleGNsdWRlcyI6IHt9CiAgICB9CiAgXQp9Cg==
            mode: 420
            overwrite: true
            path: /var/lib/kubelet/seccomp/seccomp-ls.json
    EOF
    ~~~

2. Once the MC is applied to our workers, we can try to run a pod using the new seccomp profile we created.

    1. Create a new namespace for running our tests:

        ~~~sh
        NAMESPACE=test-seccomp
        oc create ns ${NAMESPACE}
        ~~~

    2. Create a user and give it edit role permissions on the namespace

        ~~~sh
        oc -n ${NAMESPACE} create sa testuser
        oc -n ${NAMESPACE} adm policy add-role-to-user edit system:serviceaccount:${NAMESPACE}:testuser
        ~~~

    3. We are going to create our own SCC based on the restricted-v2 one, and on top of that we need to allow using this new seccomp profile.

        ~~~sh
        cat <<EOF | oc create -f -
        apiVersion: security.openshift.io/v1
        kind: SecurityContextConstraints
        metadata:
          name: restricted-v2-seccomp
        allowHostDirVolumePlugin: false
        allowHostIPC: false
        allowHostNetwork: false
        allowHostPID: false
        allowHostPorts: false
        allowPrivilegeEscalation: false
        allowPrivilegedContainer: false
        allowedCapabilities:
        - NET_BIND_SERVICE
        defaultAddCapabilities: null
        fsGroup:
          type: MustRunAs
        groups: []
        priority: null
        readOnlyRootFilesystem: false
        requiredDropCapabilities:
        - ALL
        runAsUser:
          type: MustRunAsRange
        seLinuxContext:
          type: MustRunAs
        seccompProfiles:
        - runtime/default
        - localhost/seccomp-ls.json
        supplementalGroups:
          type: RunAsAny
        users: []
        volumes:
        - configMap
        - downwardAPI
        - emptyDir
        - persistentVolumeClaim
        - projected
        - secret
        EOF
        ~~~

    4. We're granting use privileges of this new SCC to the SA test-user

        ~~~sh
        oc -n ${NAMESPACE} adm policy add-scc-to-user restricted-v2-seccomp system:serviceaccount:${NAMESPACE}:testuser
        ~~~

    5. Configure seccomp at pod level:

        >**NOTE**: Keep in mind that the runtime might need more privileges than the allowed by your seccomp profile to set up the pod infrastructure. That may lead to the pod failing to initialize. Is it recommended assigning the seccomp profiles to your containers rather than to the pod.

        ~~~sh
        cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
        apiVersion: v1
        kind: Pod
        metadata:
          name: seccomp-ls-test
        spec:
          serviceAccountName: testuser
          securityContext:
            seccompProfile:
              type: Localhost
              localhostProfile: seccomp-ls.json
          containers:
          - image: quay.io/fedora/fedora:36
            name: seccomp-ls-test
            command: ["ls", "/"]
          dnsPolicy: ClusterFirst
          restartPolicy: Never
        status: {}
        EOF
        ~~~

    6. In this case, the pod was executed without issues. But, as said before, assigning the seccomp to a container is usually a better move. Let's remove the pod and assign the seccomp profile at the container level:

        ~~~sh
        oc -n ${NAMESPACE} delete pod seccomp-ls-test
        ~~~

    7. Configure seccomp at container level:

        ~~~sh
        cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
        apiVersion: v1
        kind: Pod
        metadata:
          name: seccomp-ls-test
        spec:
          serviceAccountName: testuser
          containers:
          - image: quay.io/fedora/fedora:36
            name: seccomp-ls-test
            command: ["ls", "/"]
            securityContext:
              seccompProfile:
                type: Localhost
                localhostProfile: seccomp-ls.json
          dnsPolicy: ClusterFirst
          restartPolicy: Never
        status: {}
        EOF
        ~~~

    8. The pod has completed successfully:

        ~~~sh
        oc -n ${NAMESPACE} get pods
        NAME                        READY   STATUS      RESTARTS   AGE
        seccomp-ls-test             0/1     Completed   0          44s
        ~~~

    9. Let's try to run `ls -l /` and see if the profile block the execution:

        ~~~sh
        cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
        apiVersion: v1
        kind: Pod
        metadata:
          name: seccomp-lsl-test
          annotations:
            container.seccomp.security.alpha.kubernetes.io/seccomp-lsl-test: localhost/seccomp-ls.json
        spec:
          serviceAccountName: testuser
          containers:
          - image: quay.io/fedora/fedora:36
            name: seccomp-lsl-test
            command: ["ls", "-l", "/"]
            securityContext:
              seccompProfile:
                type: Localhost
                localhostProfile: seccomp-ls.json
          dnsPolicy: ClusterFirst
          restartPolicy: Never
        status: {}
        EOF
        ~~~

    10. As expected, the execution was blocked:

        ~~~sh
        oc -n ${NAMESPACE} logs seccomp-lsl-test
        ls: /: Operation not permitted
        ~~~

### **Auditing Seccomp Profiles**

Seccomp profiles can have different _actions_ that will cause the policy to act differently against a set of syscalls:

* **SCMP_ACT_ALLOW**: Allows the use of the specified syscalls.
* **SCMP_ACT_ERRNO**: Denies the use of the specified syscalls.
* **SCMP_ACT_LOG**: Allows the use of any syscall, but logs the ones that are not specifically permitted.

On top of the action applied to a set of syscalls, we can define a _defaultAction_ that will be applied to any syscall not present in our profile. We are going to see how to use `SCMP_ACT_LOG` action in order to audit missing syscalls in our seccomp profile.

1. In the previous lab, we were not able to use `ls -l`, let's put the policy in `LOG` mode to get the list of blocked syscalls preventing the container to work properly.

    > **NOTE**: Below patch changes `"defaultAction": "SCMP_ACT_ERRNO"` to `"defaultAction": "SCMP_ACT_LOG"` in our policy.

    ~~~sh
    oc patch mc 99-seccomp-profile-ls -p '{"spec":{"config":{"storage":{"files":[{"contents":{"source":"data:text/plain;charset=utf-8;base64,ewogICJkZWZhdWx0QWN0aW9uIjogIlNDTVBfQUNUX0xPRyIsCiAgImFyY2hpdGVjdHVyZXMiOiBbCiAgICAiU0NNUF9BUkNIX1g4Nl82NCIKICBdLAogICJzeXNjYWxscyI6IFsKICAgIHsKICAgICAgIm5hbWVzIjogWwogICAgICAgICJhY2Nlc3MiLAogICAgICAgICJhcmNoX3ByY3RsIiwKICAgICAgICAiYnJrIiwKICAgICAgICAiY2FwZ2V0IiwKICAgICAgICAiY2Fwc2V0IiwKICAgICAgICAiY2hkaXIiLAogICAgICAgICJjbG9zZSIsCiAgICAgICAgImR1cDMiLAogICAgICAgICJlcG9sbF9jdGwiLAogICAgICAgICJlcG9sbF9wd2FpdCIsCiAgICAgICAgImV4ZWN2ZSIsCiAgICAgICAgImV4aXRfZ3JvdXAiLAogICAgICAgICJmY2hkaXIiLAogICAgICAgICJmY2hvd24iLAogICAgICAgICJmY250bCIsCiAgICAgICAgImZzdGF0IiwKICAgICAgICAiZnN0YXRmcyIsCiAgICAgICAgImZ1dGV4IiwKICAgICAgICAiZ2V0ZGVudHM2NCIsCiAgICAgICAgImdldHBpZCIsCiAgICAgICAgImdldHBwaWQiLAogICAgICAgICJnZXRyYW5kb20iLAogICAgICAgICJpb2N0bCIsCiAgICAgICAgIm1tYXAiLAogICAgICAgICJtb3VudCIsCiAgICAgICAgIm1wcm90ZWN0IiwKICAgICAgICAibXVubWFwIiwKICAgICAgICAibmFub3NsZWVwIiwKICAgICAgICAibmV3ZnN0YXRhdCIsCiAgICAgICAgIm9wZW5hdCIsCiAgICAgICAgInBpcGUyIiwKICAgICAgICAicGl2b3Rfcm9vdCIsCiAgICAgICAgInByY3RsIiwKICAgICAgICAicHJlYWQ2NCIsCiAgICAgICAgInBybGltaXQ2NCIsCiAgICAgICAgInJlYWQiLAogICAgICAgICJyc2VxIiwKICAgICAgICAicnRfc2lncmV0dXJuIiwKICAgICAgICAic2VjY29tcCIsCiAgICAgICAgInNldF9yb2J1c3RfbGlzdCIsCiAgICAgICAgInNldF90aWRfYWRkcmVzcyIsCiAgICAgICAgInNldGdpZCIsCiAgICAgICAgInNldGdyb3VwcyIsCiAgICAgICAgInNldGhvc3RuYW1lIiwKICAgICAgICAic2V0dWlkIiwKICAgICAgICAic3RhdGZzIiwKICAgICAgICAic3RhdHgiLAogICAgICAgICJ0Z2tpbGwiLAogICAgICAgICJ1bWFzayIsCiAgICAgICAgInVtb3VudDIiLAogICAgICAgICJ3cml0ZSIKICAgICAgXSwKICAgICAgImFjdGlvbiI6ICJTQ01QX0FDVF9BTExPVyIsCiAgICAgICJhcmdzIjogW10sCiAgICAgICJjb21tZW50IjogIiIsCiAgICAgICJpbmNsdWRlcyI6IHt9LAogICAgICAiZXhjbHVkZXMiOiB7fQogICAgfQogIF0KfQo="},"mode":420,"overwrite":true,"path":"/var/lib/kubelet/seccomp/seccomp-ls.json"}]}}}}' --type merge
    ~~~

2. We need to wait for the new profile to be deployed on the nodes, when the updated profile is ready, we can run the pod again:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: seccomp-lsl-test-2
    spec:
      serviceAccountName: testuser
      containers:
      - image: quay.io/fedora/fedora:36
        name: seccomp-lsl-test
        command: ["ls", "-l", "/"]
        securityContext:
          seccompProfile:
            type: Localhost
            localhostProfile: seccomp-ls.json
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~

3. The pod worked since we are allowing all syscalls and logging the ones that are not explicitly permitted. In the `audit.log` from the node that executed the pod we can find the extra syscalls we need to add to our profile:

    1. Connect to the node that executed the container

        ~~~sh
        # Get the node that executed the pod
        POD_NODE=$(oc -n ${NAMESPACE} get pod seccomp-lsl-test-2 -o jsonpath='{.spec.nodeName}')
        # Get the audit log from that node and filter by comm="ls"
        oc adm node-logs ${POD_NODE} --path='audit/audit.log' | grep 'comm="ls"'
        ~~~

    2. Get the audit logs for your container

        ~~~sh
        <OUTPUT_OMITTED>
        type=SECCOMP msg=audit(1612433809.320:166): auid=4294967295 uid=1000660000 gid=0 ses=4294967295 subj=system_u:system_r:container_t:s0:c5,c26 pid=4849 comm="ls" exe="/usr/bin/coreutils" sig=0 arch=c000003e syscall=192 compat=0 ip=0x7fe33721ad5e code=0x7ffc0000AUID="unset" UID="unknown(1000660000)" GID="root" ARCH=x86_64 SYSCALL=lgetxattr
        type=SECCOMP msg=audit(1612433809.320:167): auid=4294967295 uid=1000660000 gid=0 ses=4294967295 subj=system_u:system_r:container_t:s0:c5,c26 pid=4849 comm="ls" exe="/usr/bin/coreutils" sig=0 arch=c000003e syscall=191 compat=0 ip=0x7fe33721acfe code=0x7ffc0000AUID="unset" UID="unknown(1000660000)" GID="root" ARCH=x86_64 SYSCALL=getxattr
        type=SECCOMP msg=audit(1612433809.320:168): auid=4294967295 uid=1000660000 gid=0 ses=4294967295 subj=system_u:system_r:container_t:s0:c5,c26 pid=4849 comm="ls" exe="/usr/bin/coreutils" sig=0 arch=c000003e syscall=191 compat=0 ip=0x7fe33721acfe code=0x7ffc0000AUID="unset" UID="unknown(1000660000)" GID="root" ARCH=x86_64 SYSCALL=getxattr
        type=SECCOMP msg=audit(1612433809.339:169): auid=4294967295 uid=1000660000 gid=0 ses=4294967295 subj=system_u:system_r:container_t:s0:c5,c26 pid=4849 comm="ls" exe="/usr/bin/coreutils" sig=0 arch=c000003e syscall=8 compat=0 ip=0x7fe33720d94b code=0x7ffc0000AUID="unset" UID="unknown(1000660000)" GID="root" ARCH=x86_64 SYSCALL=lseek
        ~~~

    3. With the id from the `msg` (1612433809 in this case), get all the audit lines and filter by syscall removing duplicates

        ~~~sh
        oc adm node-logs ${POD_NODE} --path='audit/audit.log' | grep 1612433809 | awk -F "SYSCALL=" '{print $2}' | sort -u
        ~~~

        ~~~sh
        getxattr
        lgetxattr
        lseek
        readlink
        ~~~

    4. This new syscalls need to be added to our seccomp profile, these are required by the container to run `ls -l`.

4. Now that we have the extra syscalls that need to be added, let's update our seccomp profile:

    > **NOTE**: We are adding the required syscalls and on top of that we changed back the **defaultAction** from _SCMP_ACT_LOG_ to _SCMP_ACT_ERRNO_.

    ~~~sh
    oc patch mc 99-seccomp-profile-ls -p '{"spec":{"config":{"storage":{"files":[{"contents":{"source":"data:text/plain;charset=utf-8;base64,ewogICJkZWZhdWx0QWN0aW9uIjogIlNDTVBfQUNUX0VSUk5PIiwKICAiYXJjaGl0ZWN0dXJlcyI6IFsKICAgICJTQ01QX0FSQ0hfWDg2XzY0IgogIF0sCiAgInN5c2NhbGxzIjogWwogICAgewogICAgICAibmFtZXMiOiBbCiAgICAgICAgImFjY2VzcyIsCiAgICAgICAgImFyY2hfcHJjdGwiLAogICAgICAgICJicmsiLAogICAgICAgICJjYXBnZXQiLAogICAgICAgICJjYXBzZXQiLAogICAgICAgICJjaGRpciIsCiAgICAgICAgImNsb3NlIiwKICAgICAgICAiZHVwMyIsCiAgICAgICAgImVwb2xsX2N0bCIsCiAgICAgICAgImVwb2xsX3B3YWl0IiwKICAgICAgICAiZXhlY3ZlIiwKICAgICAgICAiZXhpdF9ncm91cCIsCiAgICAgICAgImZjaGRpciIsCiAgICAgICAgImZjaG93biIsCiAgICAgICAgImZjbnRsIiwKICAgICAgICAiZnN0YXQiLAogICAgICAgICJmc3RhdGZzIiwKICAgICAgICAiZnV0ZXgiLAogICAgICAgICJnZXRkZW50czY0IiwKICAgICAgICAiZ2V0cGlkIiwKICAgICAgICAiZ2V0cHBpZCIsCiAgICAgICAgImdldHJhbmRvbSIsCiAgICAgICAgImlvY3RsIiwKICAgICAgICAibW1hcCIsCiAgICAgICAgIm1vdW50IiwKICAgICAgICAibXByb3RlY3QiLAogICAgICAgICJtdW5tYXAiLAogICAgICAgICJuYW5vc2xlZXAiLAogICAgICAgICJuZXdmc3RhdGF0IiwKICAgICAgICAib3BlbmF0IiwKICAgICAgICAicGlwZTIiLAogICAgICAgICJwaXZvdF9yb290IiwKICAgICAgICAicHJjdGwiLAogICAgICAgICJwcmVhZDY0IiwKICAgICAgICAicHJsaW1pdDY0IiwKICAgICAgICAicmVhZCIsCiAgICAgICAgInJzZXEiLAogICAgICAgICJydF9zaWdyZXR1cm4iLAogICAgICAgICJzZWNjb21wIiwKICAgICAgICAic2V0X3JvYnVzdF9saXN0IiwKICAgICAgICAic2V0X3RpZF9hZGRyZXNzIiwKICAgICAgICAic2V0Z2lkIiwKICAgICAgICAic2V0Z3JvdXBzIiwKICAgICAgICAic2V0aG9zdG5hbWUiLAogICAgICAgICJzZXR1aWQiLAogICAgICAgICJzdGF0ZnMiLAogICAgICAgICJzdGF0eCIsCiAgICAgICAgInRna2lsbCIsCiAgICAgICAgInVtYXNrIiwKICAgICAgICAidW1vdW50MiIsCiAgICAgICAgIndyaXRlIiwKICAgICAgICAiZ2V0eGF0dHIiLAogICAgICAgICJsZ2V0eGF0dHIiLAogICAgICAgICJsc2VlayIsCiAgICAgICAgInJlYWRsaW5rIgogICAgICBdLAogICAgICAiYWN0aW9uIjogIlNDTVBfQUNUX0FMTE9XIiwKICAgICAgImFyZ3MiOiBbXSwKICAgICAgImNvbW1lbnQiOiAiIiwKICAgICAgImluY2x1ZGVzIjoge30sCiAgICAgICJleGNsdWRlcyI6IHt9CiAgICB9CiAgXQp9Cg=="},"mode":420,"overwrite":true,"path":"/var/lib/kubelet/seccomp/seccomp-ls.json"}]}}}}' --type merge
    ~~~

5. Let's execute the pod now and see the result:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: seccomp-lsl-test-3
    spec:
      serviceAccountName: testuser
      containers:
      - image: quay.io/fedora/fedora:36
        name: seccomp-lsl-test
        command: ["ls", "-l", "/"]
        securityContext:
          seccompProfile:
            type: Localhost
            localhostProfile: seccomp-ls.json
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~

6. The pod has completed successfully, let's try to run `ping -c4 1.1.1.1` and see if the profile block the execution:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: seccomp-ping-test
    spec:
      serviceAccountName: testuser
      containers:
      - image: quay.io/mavazque/trbsht:latest
        name: seccomp-ping-test
        command: ["ping", "-c4", "1.1.1.1"]
        securityContext:
          seccompProfile:
            type: Localhost
            localhostProfile: seccomp-ls.json
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~

    >**NOTE**: As expected, the profile blocks the execution.

    ~~~sh
    oc -n ${NAMESPACE} logs seccomp-ping-test
    
    ping: setuid: Invalid argument
    ~~~

### **Using the Security Profiles Operator**

So far we have seen how to create seccomp profiles using the oci hook or by changing the profile to audit mode. In this section we're going to introduce the [Security Profiles Operator](https://github.com/kubernetes-sigs/security-profiles-operator), an operator that help us manage the seccomp profiles in our cluster and that can also be used to create seccomp profiles out of a given workload.

Following the last example in the previous section, we will deploy the Security profiles Operator and create a recording for the ping workload.

There are multiple ways to deploy the operator, we will use the manual approach. You can read more about installation [here](https://github.com/kubernetes-sigs/security-profiles-operator/blob/main/installation-usage.md)

1. Deploy the Operator:

    ~~~sh
    oc apply -f https://raw.githubusercontent.com/kubernetes-sigs/security-profiles-operator/v0.4.3/deploy/operator.yaml
    oc -n security-profiles-operator wait --for condition=ready pod -l name=security-profiles-operator
    oc -n security-profiles-operator wait --for condition=ready pod -l name=security-profiles-operator-webhook
    oc -n security-profiles-operator wait --for condition=ready pod -l name=spod
    ~~~

2. Enable the log enricher to enable profile recording using the logs recorder:

    ~~~sh
    # If the patch below fails saying server doesn't have a resource type spod, wait a bit and run again the patch
    oc -n security-profiles-operator patch spod spod --type=merge -p '{"spec":{"enableLogEnricher":true}}'
    oc -n security-profiles-operator wait --for condition=ready pod -l name=spod
    ~~~

3. Now that we have the operator running, let's remove the ping pod from the previous section:

    ~~~sh
    oc -n ${NAMESPACE} delete pod seccomp-ping-test
    ~~~

4. Let's create a recording for our application:

    > **NOTE**: We need to use the matchLabels selector to read the syscalls used by pods using this label.

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create -f -
    apiVersion: security-profiles-operator.x-k8s.io/v1alpha1
    kind: ProfileRecording
    metadata:
      name: ping
    spec:
      kind: SeccompProfile
      recorder: logs
      podSelector:
        matchLabels:
          app: seccomp-ping-test
    EOF
    ~~~

5. Now that we have the recording created, let's create the ping workload:

    > **NOTE**: We're creating the workload without any seccomp profile assigned.

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: seccomp-ping-test
      labels:
        app: seccomp-ping-test
    spec:
      serviceAccountName: testuser
      containers:
      - image: quay.io/mavazque/trbsht:latest
        name: seccomp-ping-test
        command: ["ping", "-c4", "1.1.1.1"]
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~

6. The profile recording should be targeting our workload:

    ~~~sh
    oc -n ${NAMESPACE} get profilerecording ping -o jsonpath='{.status}' | jq
    ~~~

    ~~~sh
    {
      "activeWorkloads": [
        "seccomp-ping-test"
      ]
    }
    ~~~

7. Now that our workload is completed we can delete the ping pod:

    > **NOTE**: When creating a profile for a workload, make sure that you go thought all the possible scenarios handled by your app, otherwise you may miss some syscalls required for your app to work.

    ~~~sh
    oc -n ${NAMESPACE} delete pod seccomp-ping-test
    ~~~

8. The operator handles the installation of the seccomp profile, let's check the profiles available in the cluster:

    ~~~sh
    oc get seccompprofile -A
    ~~~

    ~~~sh
    NAMESPACE                    NAME                     STATUS      AGE
    security-profiles-operator   log-enricher-trace       Installed   14m
    security-profiles-operator   nginx-1.19.1             Installed   14m
    test-seccomp                 ping-seccomp-ping-test   Installed   2m50s
    ~~~

9. We have the `ping-seccomp-ping-test` installed. One cool feature of the operator is that it handles the installation of new profiles without having to add them to a MachineConfig and wait for the nodes to reboot. We can delete the recording now:

    ~~~sh
    oc -n ${NAMESPACE} delete profilerecording ping
    ~~~

10. Let's use the new profile in our workload:

    1. We need to allow the usage of this new seccomp profile in our SCC:

        ~~~sh
        oc patch scc restricted-v2-seccomp -p "{\"seccompProfiles\":[\"runtime/default\",\"localhost/seccomp-ls.json\",\"localhost/operator/${NAMESPACE}/ping-seccomp-ping-test.json\"]}" --type merge
        ~~~

    2. Create the workload using the new profile:

        ~~~sh
        cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
        apiVersion: v1
        kind: Pod
        metadata:
          name: seccomp-ping-test
        spec:
          securityContext:
            seccompProfile:
              type: Localhost
              localhostProfile: operator/${NAMESPACE}/ping-seccomp-ping-test.json
          serviceAccountName: testuser
          containers:
          - image: quay.io/mavazque/trbsht:latest
            name: seccomp-ping-test
            command: ["ping", "-c4", "1.1.1.1"]
          dnsPolicy: ClusterFirst
          restartPolicy: Never
        status: {}
        EOF
        ~~~

    3. This time it worked:

        ~~~sh
        oc -n ${NAMESPACE} logs seccomp-ping-test

        PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
        64 bytes from 1.1.1.1: icmp_seq=1 ttl=39 time=28.3 ms
        64 bytes from 1.1.1.1: icmp_seq=2 ttl=39 time=27.2 ms
        64 bytes from 1.1.1.1: icmp_seq=3 ttl=39 time=26.7 ms
        64 bytes from 1.1.1.1: icmp_seq=4 ttl=39 time=26.9 ms

        --- 1.1.1.1 ping statistics ---
        4 packets transmitted, 4 received, 0% packet loss, time 3005ms
        rtt min/avg/max/mdev = 26.714/27.266/28.276/0.609 ms
        ~~~

As you have seen this operator eases the creation of new profiles via its recording capabilities, the operator can also manage the deployment of new profiles to the cluster, you can see an example below:

~~~yaml
apiVersion: security-profiles-operator.x-k8s.io/v1beta1
kind: SeccompProfile
metadata:
  name: ls
spec:
  architectures:
  - SCMP_ARCH_X86_64
  - SCMP_ARCH_X86
  - SCMP_ARCH_X32
  defaultAction: SCMP_ACT_ERRNO
  syscalls:
  - action: SCMP_ACT_ALLOW
    names:
    - access
    - arch_prctl
    - brk
    - capget
    - capset
    - chdir
    - close
    - dup3
    - epoll_ctl
    - epoll_pwait
    - execve
    - exit_group
    - fchdir
    - fcntl
    - fstat
    - fstatfs
    - futex
    - getdents64
    - getpid
    - getppid
    - getrandom
    - ioctl
    - mmap
    - mount
    - mprotect
    - munmap
    - nanosleep
    - newfstatat
    - openat
    - pipe2
    - pivot_root
    - prctl
    - pread64
    - prlimit64
    - read
    - rseq
    - rt_sigreturn
    - set_robust_list
    - set_tid_address
    - setgid
    - setgroups
    - sethostname
    - setuid
    - statfs
    - statx
    - tgkill
    - umask
    - umount2
    - write
~~~