# **Secccomp Security Profiles**

In this demo we will learn how we can work with seccomp profiles for our containers.

We will start by generating a seccomp profile for a local container with podman, and then we will see how we can use that profile for pods running on OpenShift.

## **Creating a Seccomp Profile with Podman**

In order to generate our `seccomp` profile we are going to rely on an OCI Hook which will intercept the syscalls made by our container in order to get them added to the seccomp profile.

The OCI Hook project can be found here: https://github.com/containers/oci-seccomp-bpf-hook

1. Create a VM with Centos 8 Streams to run our tests. I'm using [kcli](https://github.com/karmab/kcli).

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
4. Let's run a test container that runs an ls command

    ~~~sh
    sudo podman run --rm --runtime /usr/bin/runc --annotation io.containers.trace-syscall="of:/tmp/ls.json" fedora:32 ls / > /dev/null
    ~~~
5. The hook has generated the required `seccomp` profile at `/tmp/ls.json` let's review it.
    
    1. You can see the syscalls that ls needed in order to list the / filesystem
        
        ~~~sh
        cat /tmp/ls.json | jq '.syscalls[0].names'
        ~~~
6. Now let's try to use this `seccomp` profile to run the container

    ~~~sh
    sudo podman run --rm --runtime /usr/bin/runc --security-opt seccomp=/tmp/ls.json fedora:32 ls /
    ~~~
7. The container is now running which the minimum posible set of syscall allowed, let's try to change the ls command a bit:

    ~~~sh
    sudo podman run --rm --runtime /usr/bin/runc --security-opt seccomp=/tmp/ls.json fedora:32 ls -l /
   
    ~~~

    ~~~sh
    ls: cannot access '/': Operation not permitted
    ~~~
    
    1. It seems that ls usses additional syscalls with the -l parameter. It's very important that you do a exhaustive testing of your application when tracing the syscalls required by it. Otherwise you will find issues similat to this.
8. The hook allow us to pass an input file that will be used as baseline, then we will log the required additional syscalls into a new output file:

    ~~~sh
    sudo podman run --rm --runtime /usr/bin/runc --annotation io.containers.trace-syscall="if:/tmp/ls.json;of:/tmp/lsl.json" fedora:32 ls -l / > /dev/null
    ~~~
9. Now we should have a valid seccomp profile at `/tmp/lsl.json` that we can use for running our container:

    ~~~sh
    sudo podman run --rm --runtime /usr/bin/runc --security-opt seccomp=/tmp/lsl.json fedora:32 ls -l /
    total 4
    lrwxrwxrwx.   1 root root    7 Jan 28  2020 bin -> usr/bin
    dr-xr-xr-x.   2 root root    6 Jan 28  2020 boot
    drwxr-xr-x.   5 root root  340 Feb 16 09:15 dev
    drwxr-xr-x.  47 root root 4096 Jan  6 06:48 etc
    drwxr-xr-x.   2 root root    6 Jan 28  2020 home
    lrwxrwxrwx.   1 root root    7 Jan 28  2020 lib -> usr/lib
    lrwxrwxrwx.   1 root root    9 Jan 28  2020 lib64 -> usr/lib64
    drwx------.   2 root root    6 Jan  6 06:47 lost+found
    drwxr-xr-x.   2 root root    6 Jan 28  2020 media
    drwxr-xr-x.   2 root root    6 Jan 28  2020 mnt
    drwxr-xr-x.   2 root root    6 Jan 28  2020 opt
    ...
    ~~~

## **Using Seccomp Profiles on OpenShift**

At this point we have a valid seccomp profile, how can we get it added to OpenShift and start using it in our workloads? - Let's see.

### **Using the seccomp profile in a workload**

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
              source: data:text/plain;charset=utf-8;base64,ewogICJkZWZhdWx0QWN0aW9uIjogIlNDTVBfQUNUX0VSUk5PIiwKICAiYXJjaGl0ZWN0dXJlcyI6IFsKICAgICJTQ01QX0FSQ0hfWDg2XzY0IgogIF0sCiAgInN5c2NhbGxzIjogWwogICAgewogICAgICAibmFtZXMiOiBbCiAgICAgICAgImFjY2VzcyIsCiAgICAgICAgImFyY2hfcHJjdGwiLAogICAgICAgICJicmsiLAogICAgICAgICJjYXBnZXQiLAogICAgICAgICJjYXBzZXQiLAogICAgICAgICJjaGRpciIsCiAgICAgICAgImNsb3NlIiwKICAgICAgICAiZXBvbGxfY3RsIiwKICAgICAgICAiZXBvbGxfcHdhaXQiLAogICAgICAgICJleGVjdmUiLAogICAgICAgICJleGl0X2dyb3VwIiwKICAgICAgICAiZmNob3duIiwKICAgICAgICAiZmNudGwiLAogICAgICAgICJmc3RhdCIsCiAgICAgICAgImZzdGF0ZnMiLAogICAgICAgICJmdXRleCIsCiAgICAgICAgImdldGRlbnRzNjQiLAogICAgICAgICJnZXRwaWQiLAogICAgICAgICJnZXRwcGlkIiwKICAgICAgICAiaW9jdGwiLAogICAgICAgICJtbWFwIiwKICAgICAgICAibXByb3RlY3QiLAogICAgICAgICJtdW5tYXAiLAogICAgICAgICJuYW5vc2xlZXAiLAogICAgICAgICJuZXdmc3RhdGF0IiwKICAgICAgICAib3BlbmF0IiwKICAgICAgICAicHJjdGwiLAogICAgICAgICJwcmVhZDY0IiwKICAgICAgICAicHJsaW1pdDY0IiwKICAgICAgICAicmVhZCIsCiAgICAgICAgInJ0X3NpZ2FjdGlvbiIsCiAgICAgICAgInJ0X3NpZ3Byb2NtYXNrIiwKICAgICAgICAicnRfc2lncmV0dXJuIiwKICAgICAgICAic2NoZWRfeWllbGQiLAogICAgICAgICJzZWNjb21wIiwKICAgICAgICAic2V0X3JvYnVzdF9saXN0IiwKICAgICAgICAic2V0X3RpZF9hZGRyZXNzIiwKICAgICAgICAic2V0Z2lkIiwKICAgICAgICAic2V0Z3JvdXBzIiwKICAgICAgICAic2V0dWlkIiwKICAgICAgICAic3RhdCIsCiAgICAgICAgInN0YXRmcyIsCiAgICAgICAgInRna2lsbCIsCiAgICAgICAgIndyaXRlIgogICAgICBdLAogICAgICAiYWN0aW9uIjogIlNDTVBfQUNUX0FMTE9XIiwKICAgICAgImFyZ3MiOiBbXSwKICAgICAgImNvbW1lbnQiOiAiIiwKICAgICAgImluY2x1ZGVzIjoge30sCiAgICAgICJleGNsdWRlcyI6IHt9CiAgICB9CiAgXQp9Cg==
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
    3. We are going to create our own SCC based on the restricted one, and on top of that we need to allow using this new seccomp profile.
        
        > **NOTE**: We removed `system:authenticated` group so we will need to assign the SCC manually to our SAs/Users/Groups.

        ~~~sh
        cat <<EOF | oc create -f -
        kind: SecurityContextConstraints
        metadata:
          name: restricted-seccomp
        priority: null
        readOnlyRootFilesystem: false
        requiredDropCapabilities:
        - KILL
        - MKNOD
        - SETUID
        - SETGID
        runAsUser:
          type: MustRunAsRange
        seLinuxContext:
          type: MustRunAs
        seccompProfiles:
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
        allowHostDirVolumePlugin: false
        allowHostIPC: false
        allowHostNetwork: false
        allowHostPID: false
        allowHostPorts: false
        allowPrivilegeEscalation: true
        allowPrivilegedContainer: false
        allowedCapabilities: null
        apiVersion: security.openshift.io/v1
        defaultAddCapabilities: null
        fsGroup:
          type: MustRunAs
        groups: []
        EOF
        ~~~
    4. We're granting use privileges of this new SCC to the SA test-user

        ~~~sh
        oc -n ${NAMESPACE} adm policy add-scc-to-user restricted-seccomp system:serviceaccount:${NAMESPACE}:testuser
        ~~~
    5. Configure seccomp at pod level:

        >**NOTE**: Keep in mind that the runtime might need more privileges than the allowed by your seccomp profile to setup the pod infrastructure. That may lead to the pod failing to initialize. Is it recommended to assign the seccomp profiles to your containers rather than to the pod.
        
        ~~~sh
        cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
        apiVersion: v1
        kind: Pod
        metadata:
          name: seccomp-ls-test
          annotations:
            seccomp.security.alpha.kubernetes.io/pod: localhost/seccomp-ls.json
        spec:
          serviceAccountName: testuser
          securityContext:
            seccompProfile:
              type: Localhost
              localhostProfile: seccomp-ls.json
          containers:
          - image: registry.centos.org/centos:8
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
          annotations:
            container.seccomp.security.alpha.kubernetes.io/seccomp-ls-test: localhost/seccomp-ls.json
        spec:
          serviceAccountName: testuser
          containers:
          - image: registry.centos.org/centos:8
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
    8. The pod has completed succesfully:
    
        ~~~sh
        oc get pods
        NAME                        READY   STATUS      RESTARTS   AGE
        seccomp-ls-test             0/1     Completed   0          44s
        ~~~
    
    Let's try to run `ls -l /` and see if the profile block the execution:

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
          - image: registry.centos.org/centos:8
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
    9. As expected, the execution was blocked:

        ~~~sh
        oc -n ${NAMESPACE} logs seccomp-lsl-test
    
        ls: cannot access '/': Operation not permitted
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
    oc patch mc 99-seccomp-profile-ls -p '{"spec":{"config":{"storage":{"files":[{"contents":{"source":"data:text/plain;charset=utf-8;base64,ewogICJkZWZhdWx0QWN0aW9uIjogIlNDTVBfQUNUX0xPRyIsCiAgImFyY2hpdGVjdHVyZXMiOiBbCiAgICAiU0NNUF9BUkNIX1g4Nl82NCIKICBdLAogICJzeXNjYWxscyI6IFsKICAgIHsKICAgICAgIm5hbWVzIjogWwogICAgICAgICJhY2Nlc3MiLAogICAgICAgICJhcmNoX3ByY3RsIiwKICAgICAgICAiYnJrIiwKICAgICAgICAiY2FwZ2V0IiwKICAgICAgICAiY2Fwc2V0IiwKICAgICAgICAiY2hkaXIiLAogICAgICAgICJjbG9zZSIsCiAgICAgICAgImVwb2xsX2N0bCIsCiAgICAgICAgImVwb2xsX3B3YWl0IiwKICAgICAgICAiZXhlY3ZlIiwKICAgICAgICAiZXhpdF9ncm91cCIsCiAgICAgICAgImZjaG93biIsCiAgICAgICAgImZjbnRsIiwKICAgICAgICAiZnN0YXQiLAogICAgICAgICJmc3RhdGZzIiwKICAgICAgICAiZnV0ZXgiLAogICAgICAgICJnZXRkZW50czY0IiwKICAgICAgICAiZ2V0cGlkIiwKICAgICAgICAiZ2V0cHBpZCIsCiAgICAgICAgImlvY3RsIiwKICAgICAgICAibW1hcCIsCiAgICAgICAgIm1wcm90ZWN0IiwKICAgICAgICAibXVubWFwIiwKICAgICAgICAibmFub3NsZWVwIiwKICAgICAgICAibmV3ZnN0YXRhdCIsCiAgICAgICAgIm9wZW5hdCIsCiAgICAgICAgInByY3RsIiwKICAgICAgICAicHJlYWQ2NCIsCiAgICAgICAgInBybGltaXQ2NCIsCiAgICAgICAgInJlYWQiLAogICAgICAgICJydF9zaWdhY3Rpb24iLAogICAgICAgICJydF9zaWdwcm9jbWFzayIsCiAgICAgICAgInJ0X3NpZ3JldHVybiIsCiAgICAgICAgInNjaGVkX3lpZWxkIiwKICAgICAgICAic2VjY29tcCIsCiAgICAgICAgInNldF9yb2J1c3RfbGlzdCIsCiAgICAgICAgInNldF90aWRfYWRkcmVzcyIsCiAgICAgICAgInNldGdpZCIsCiAgICAgICAgInNldGdyb3VwcyIsCiAgICAgICAgInNldHVpZCIsCiAgICAgICAgInN0YXQiLAogICAgICAgICJzdGF0ZnMiLAogICAgICAgICJ0Z2tpbGwiLAogICAgICAgICJ3cml0ZSIKICAgICAgXSwKICAgICAgImFjdGlvbiI6ICJTQ01QX0FDVF9BTExPVyIsCiAgICAgICJhcmdzIjogW10sCiAgICAgICJjb21tZW50IjogIiIsCiAgICAgICJpbmNsdWRlcyI6IHt9LAogICAgICAiZXhjbHVkZXMiOiB7fQogICAgfQogIF0KfQo="},"mode":420,"overwrite":true,"path":"/var/lib/kubelet/seccomp/seccomp-ls.json"}]}}}}' --type merge
    ~~~ 
2. We need to wait for the new profile to be deployed on the nodes, when the updated profile is ready, we can run the pod again:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: seccomp-lsl-test-2
      annotations:
        container.seccomp.security.alpha.kubernetes.io/seccomp-lsl-test: localhost/seccomp-ls.json
    spec:
      serviceAccountName: testuser
      containers:
      - image: registry.centos.org/centos:8
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
        
        ~~~
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

        ~~~
        connect
        getxattr
        lgetxattr
        lseek
        lstat
        readlink
        socket
        ~~~
    4. This new syscalls need to be added to our seccomp profile, these are required by the container to run `ls -l`.

4. Now that we have the extra syscalls that need to be added, let's update our seccomp profile:

    > **NOTE**: We are adding the required syscalls and on top of that we changed back the **defaultAction** from _SCMP_ACT_LOG_ to _SCMP_ACT_ERRNO_.

    ~~~sh
    oc patch mc 99-seccomp-profile-ls -p '{"spec":{"config":{"storage":{"files":[{"contents":{"source":"data:text/plain;charset=utf-8;base64,ewogICJkZWZhdWx0QWN0aW9uIjogIlNDTVBfQUNUX0VSUk5PIiwKICAiYXJjaGl0ZWN0dXJlcyI6IFsKICAgICJTQ01QX0FSQ0hfWDg2XzY0IgogIF0sCiAgInN5c2NhbGxzIjogWwogICAgewogICAgICAibmFtZXMiOiBbCiAgICAgICAgImFjY2VzcyIsCiAgICAgICAgImFyY2hfcHJjdGwiLAogICAgICAgICJicmsiLAogICAgICAgICJjYXBnZXQiLAogICAgICAgICJjYXBzZXQiLAogICAgICAgICJjaGRpciIsCiAgICAgICAgImNsb3NlIiwKICAgICAgICAiZXBvbGxfY3RsIiwKICAgICAgICAiZXBvbGxfcHdhaXQiLAogICAgICAgICJleGVjdmUiLAogICAgICAgICJleGl0X2dyb3VwIiwKICAgICAgICAiZmNob3duIiwKICAgICAgICAiZmNudGwiLAogICAgICAgICJmc3RhdCIsCiAgICAgICAgImZzdGF0ZnMiLAogICAgICAgICJmdXRleCIsCiAgICAgICAgImdldGRlbnRzNjQiLAogICAgICAgICJnZXRwaWQiLAogICAgICAgICJnZXRwcGlkIiwKICAgICAgICAiaW9jdGwiLAogICAgICAgICJtbWFwIiwKICAgICAgICAibXByb3RlY3QiLAogICAgICAgICJtdW5tYXAiLAogICAgICAgICJuYW5vc2xlZXAiLAogICAgICAgICJuZXdmc3RhdGF0IiwKICAgICAgICAib3BlbmF0IiwKICAgICAgICAicHJjdGwiLAogICAgICAgICJwcmVhZDY0IiwKICAgICAgICAicHJsaW1pdDY0IiwKICAgICAgICAicmVhZCIsCiAgICAgICAgInJ0X3NpZ2FjdGlvbiIsCiAgICAgICAgInJ0X3NpZ3Byb2NtYXNrIiwKICAgICAgICAicnRfc2lncmV0dXJuIiwKICAgICAgICAic2NoZWRfeWllbGQiLAogICAgICAgICJzZWNjb21wIiwKICAgICAgICAic2V0X3JvYnVzdF9saXN0IiwKICAgICAgICAic2V0X3RpZF9hZGRyZXNzIiwKICAgICAgICAic2V0Z2lkIiwKICAgICAgICAic2V0Z3JvdXBzIiwKICAgICAgICAic2V0dWlkIiwKICAgICAgICAic3RhdCIsCiAgICAgICAgInN0YXRmcyIsCiAgICAgICAgInRna2lsbCIsCiAgICAgICAgIndyaXRlIiwKICAgICAgICAiY29ubmVjdCIsCiAgICAgICAgImdldHhhdHRyIiwKICAgICAgICAibGdldHhhdHRyIiwKICAgICAgICAibHNlZWsiLAogICAgICAgICJsc3RhdCIsCiAgICAgICAgInJlYWRsaW5rIiwKICAgICAgICAic29ja2V0IgogICAgICBdLAogICAgICAiYWN0aW9uIjogIlNDTVBfQUNUX0FMTE9XIiwKICAgICAgImFyZ3MiOiBbXSwKICAgICAgImNvbW1lbnQiOiAiIiwKICAgICAgImluY2x1ZGVzIjoge30sCiAgICAgICJleGNsdWRlcyI6IHt9CiAgICB9CiAgXQp9Cg=="},"mode":420,"overwrite":true,"path":"/var/lib/kubelet/seccomp/seccomp-ls.json"}]}}}}' --type merge
    ~~~

5. Let's execute the pod now and see the result:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: seccomp-lsl-test-3
      annotations:
        container.seccomp.security.alpha.kubernetes.io/seccomp-lsl-test: localhost/seccomp-ls.json
    spec:
      serviceAccountName: testuser
      containers:
      - image: registry.centos.org/centos:8
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

6. The pod has completed succesfully, let's try to run `ping -c4 1.1.1.1` and see if the profile block the execution:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: seccomp-ping-test
      annotations:
        container.seccomp.security.alpha.kubernetes.io/seccomp-ping-test: localhost/seccomp-ls.json
    spec:
      serviceAccountName: testuser
      containers:
      - image: registry.centos.org/centos:8
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
    
    setuid: Invalid argument
    ~~~
    
    It is left to the reader creating a seccomp profile that allows ping command to be executed. Remember the two options that have been shown, one it is creating an specific ping secommp list using podman and the bpf hook we already installed. Other option could be to create an empty seccomp profile defaulting to SCMP_ACT_LOG. Do not forget to add the new seccomp json profile to the cluster using the MCO and list it into the allowed seccomp profiles in the restricted-seccomp SCC.
