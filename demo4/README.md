# **SCCs Strategies**

In this demo we will learn how we can work with the different strategies for our workloads.

## **Hands-on Demo 1**

In this demo we will see a few `RunAsUser` strategies we can configure.

**MustRunAsRange Strategy**

1. Create a namespace for running the demo:

    ~~~sh
    NAMESPACE="scc-strategies"
    oc create ns ${NAMESPACE}
    ~~~
2. Create a user and give it edit role on the namespace:

    ~~~sh
    oc -n ${NAMESPACE} create sa testuser
    oc -n ${NAMESPACE} adm policy add-role-to-user edit system:serviceaccount:${NAMESPACE}:testuser
    ~~~
3. We are going to create our own SCC based on the restricted one:
        
    > **NOTE**: We removed `system:authenticated` group so we will need to assign the SCC manually to our SAs/Users/Groups. On top of that we added `priority: 1` so it has more priority than `restricted` SCC when they both are elegible.

    ~~~sh
    cat <<EOF | oc create -f -
    kind: SecurityContextConstraints
    metadata:
      name: restricted-runasuser
    priority: 1
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
4. We're granting use privileges of this new SCC to the SA test-user:

    ~~~sh
    oc -n ${NAMESPACE} adm policy add-scc-to-user restricted-runasuser system:serviceaccount:${NAMESPACE}:testuser
    ~~~
5. Our SCC has the following configuration:

    ~~~yaml
    runAsUser:
      type: MustRunAsRange
    ~~~
6. Since we're not defining a range, the availabe UID range for pods to use will be determined by the range assigned to the namespace:

    ~~~sh
    oc get ns ${NAMESPACE} -o jsonpath='{.metadata.annotations.openshift\.io\/sa\.scc\.uid-range}'
    ~~~

    > **NOTE**: The output indicates that the first UID available in our range is the `1000650000` and we have `+10000` UIDs after that first one.

    ~~~
    1000650000/10000
    ~~~
7. Let's create a deployment without explicitly telling it what UID to use:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        app: reversewords-app
      name: reversewords-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: reversewords-app
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: reversewords-app
        spec:
          serviceAccountName: testuser
          containers:
          - image: quay.io/mavazque/reversewords:ubi8
            name: reversewords
            resources: {}
    status: {}
    EOF
    ~~~
8. We can see how the pod is running with the UID `1000650000`:

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app -- whoami
    ~~~

    ~~~
    1000650000
    ~~~
9. We can now patch the deployment so we explicitly request a UID from the range:

    ~~~sh
    oc -n ${NAMESPACE} patch deployment reversewords-app -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"reversewords"}],"containers":[{"name":"reversewords","securityContext":{"runAsUser":1000650015}}]}}}}'
    ~~~
10. Now the pod will be running with the requested UID `1000650015`:

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app -- whoami
    ~~~

    ~~~
    1000650015
    ~~~
11. If we try to use a random nonroot UID outside the allowed range this is what happens:

    ~~~sh
    oc -n ${NAMESPACE} patch deployment reversewords-app -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"reversewords"}],"containers":[{"name":"reversewords","securityContext":{"runAsUser":5000}}]}}}}'
    ~~~

    ~~~sh
    oc -n ${NAMESPACE} get deployment reversewords-app -o yaml | grep -A50 ^status:
    ~~~

    > The deployment cannot create the pod because the request UID `5000` is not within the allowed range.
    ~~~
    status:
    <OUTPUT_OMITTED>
      - lastTransitionTime: "2021-02-11T18:08:50Z"
        lastUpdateTime: "2021-02-11T18:08:50Z"
        message: 'pods "reversewords-app-79d8dbf889-" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.runAsUser: Invalid value: 5000: must be in the ranges: [1000650000, 1000659999] spec.containers[0].securityContext.runAsUser: Invalid value: 5000: must be in the ranges: [1000650000, 1000659999]]'
        reason: FailedCreate
        status: "True"
        type: ReplicaFailure
    <OUTPUT_OMITTED>
    ~~~
12. If we want to provide our own valid range, we can edit the SCC in order to allow the range we choose:

    > **NOTE**: Below command patches our SCC to allow UIDs for our pods between 2000 and 2500.

    ~~~sh
    oc patch scc restricted-runasuser -p '{"runAsUser":{"uidRangeMax":2500,"uidRangeMin":2000}}' --type merge
    ~~~
13. If we patch the deployment again to remove the `runAsUser` the pod will get the first UID available in the new range:

    ~~~sh
    oc -n ${NAMESPACE} patch deployment reversewords-app -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"reversewords"}],"containers":[{"name":"reversewords","securityContext":null}]}}}}'
    ~~~

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app -- whoami
    ~~~

    ~~~
    2000
    ~~~
14. One interesting experiment is what will happen if instead of the range [2000,2500] we force the container to use an UID from the namespace range. Let's do it:

    ~~~sh
    oc -n ${NAMESPACE} patch deployment reversewords-app -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"reversewords"}],"containers":[{"name":"reversewords","securityContext":{"runAsUser":1000650010}}]}}}}'
    ~~~

We can think from the results that it is possible then to specify any of the ranges defined either on the SCC or the namespace:

    ~~~sh
     oc rsh reversewords-app-f9d86bd77-m9nsz whoami
     1000650010
    ~~~

However if we pay attention we can see that our SA can use two SCC: the custom one which has higher priority and the default one: restricted. In this case, since the custom one defines a different range than the requested, it is skipped. The restricted one matches perfectly the user requested into the range specified by the namespace so it is used:

    ~~~sh
    oc describe pod  reversewords-app-f9d86bd77-m9nsz | grep -i "openshift.io/scc:"
              openshift.io/scc: restricted
    ~~~

>**NOTE:** You can check the SCC or SCCs that can be used to fulfill the pod spec by executing the scc-review command. They result is a list of SCC ordered.


**MustRunAs Strategy**

1. We can also force all our pods to run with a specific uid, let's patch our SCC first:

    ~~~sh
    oc patch scc restricted-runasuser -p '{"runAsUser":{"type":"MustRunAs","uid":1024,"uidRangeMax":null,"uidRangeMin":null}}' --type merge
    ~~~

2. Let's force a recreation of the pod:

    ~~~sh
    oc -n ${NAMESPACE} delete pod -l app=reversewords-app
    ~~~

3. If we check the UID now, we will see that it's the one we forced `1024`:

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app -- whoami
    ~~~

    ~~~
    1024
    ~~~
4. If we try to patch the deployment to make it run with a different UID, it won't work.

**RunAsAny and MustRunAsNonRoot Strategies**

These two strategies can be used as well. `RunAsAny` allows our pods to run with any UID, including `UID 0`, whereas `MustRunAsNonRoot` allows our pods to run with any UID, but `UID 0`.

These strategies do not provide default values.

**runAsGroup**

By default (unless you specify runAsGroup) the user running the container is always a member of the root group (GID 0), which doesn't grant any privileges but allows the user to read/write files accessible by GID 0. That's required for some apps that need to read config files owned by root. We can think of an example nginx pod which has its configuration files owned by root with '0640' permission. The nginx process could be run with a random UID (which cannot read the root owned files), but since the user is a member of the root group it will be able to read the config files and run nginx properly.

## **Hands-on Demo 2**

In this demo we will se how we can use `FSGroup` and `SupplementalGroups` strategies for accesing storage from our pods.

Both strategies have the same config modes we already saw for `RunAsUser`, so instead of focusing on how the different config modes affect the user that runs the pod, we will focus on what are the differences between using `FSGroup` and `SupplementalGroups` and how they impact the storage mounted on the pod.

**FSGroup**

As you might know, `FSGroup` is mainly used for block storage because it will chown the content of the storage so the group id defined can read/write content. As you can imagine, running such operation in a shared storage is not a good idea.

We can still decide how the content is chowned using the `FSGroupChangePolicy`, but still, there are storage plugins like nfs that won't take into account `FSGroup` strategies.

* `OnRootMismatch`: Only change permissions and ownership if permission and ownership of root directory does not match with expected permissions of the volume. This could help shorten the time it takes to change ownership and permission of a volume.

* `Always`: Always change permission and ownership of the volume when volume is mounted.

In this demo we will mount a pv inside a pod and we will see how the different configurations for `FSGroup` affect to the files created on the storage.

1. First, we need a storage class to dynamically bind volumes to pods.

    ~~~sh
    cat <<EOF | oc create -f -
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: local-storage
    provisioner: kubernetes.io/no-provisioner
    volumeBindingMode: WaitForFirstConsumer
    EOF
    ~~~
2. We are going to create a `local` `PersistentVolume` which will give us access to a folder inside the `worker` nodes:

    ~~~sh
    cat <<EOF | oc create -f -
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: test-volume
    spec:
      capacity:
        storage: 5Gi
      volumeMode: Filesystem
      accessModes:
      - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: local-storage
      local:
        path: /var/tmp
      nodeAffinity:
        required:
          nodeSelectorTerms:
          - matchExpressions:
            - key: node-role.kubernetes.io/worker
              operator: In
              values:
              - ""
    EOF
    ~~~
3. Finally, we create the `PersistentVolumeClaim`:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create -f -
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: test-volume-claim
    spec:
      storageClassName: local-storage
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
    EOF
    ~~~
4. Before creating our deployment, we are going to tune the SCC we have been using in the first hands on lab. This time we will configure a GID range for the `FSGroup`, that way our containers will need to use that range for mounting the storage.

    > **NOTE**: Below patch will allow pods using our SCC to use GIDs between 5000 and 6000. If no `FSGroup` is set in the `securityContext` configuration, it will get the first GID from the range, `5000` in this case.

    ~~~sh
    oc patch scc restricted-runasuser -p '{"fsGroup":{"ranges":[{"max":6000,"min":5000}]}}' --type merge
    ~~~
5. Now, it's time to create the deployment, we will use `FSGroup` 5005:
    
    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        app: reversewords-app-storage
      name: reversewords-app-storage
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: reversewords-app-storage
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: reversewords-app-storage
        spec:
          securityContext:
            fsGroup: 5005
          volumes:
          - name: test-volume
            persistentVolumeClaim:
              claimName: test-volume-claim
          serviceAccountName: testuser
          containers:
          - image: quay.io/mavazque/reversewords:ubi8
            name: reversewords
            resources: {}
            volumeMounts:
              - name: test-volume
                mountPath: "/mnt"
    status: {}
    EOF
    ~~~
6. If we check the group assigned to our pod we will see how it got assigned the secondary group `5005`:

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- id
    ~~~

    ~~~
    uid=1024(1024) gid=0(root) groups=0(root),5005
    ~~~
7. If we check the storage we mounted we will see a couple things:

    1. The directory is owned by the configured FSGroup and has the `SETGID` bit enabled.

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- ls -ld /mnt/
        ~~~

        ~~~
        drwxrwsrwt. 3 root 5005 117 Feb 12 11:03 /mnt/
        ~~~

    2. When we create a file it gets created with our UID and GID:

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- touch /mnt/testfile
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- ls -lrt /mnt/
        ~~~

        ~~~
        total 199008
        -rw-rw-r--. 1 root 5005     13802 Jan 20 10:54 df-output
        -rw-rw-r--. 1 root 5005 203764702 Jan 20 10:55 kubelet
        drwxrws---. 3 root 5005        17 Feb  9 12:33 systemd-private-362be34de0c3408684990c1a68f76806-chronyd.service-iNYlCW
        -rw-r--r--. 1 1024 5005         0 Feb 12 11:25 testfile
        ~~~
8. We mentioned that `FSGroup` chowns the content in the volume, let's try to change the group used to run our pod:

    1. Scale the deployment to 0:
        
        ~~~sh
        oc -n ${NAMESPACE} scale deployment reversewords-app-storage --replicas 0
        ~~~
    2. Patch the deployment so it uses the new group, `6000` this time:

        ~~~sh
        oc -n ${NAMESPACE} patch deployment reversewords-app-storage -p '{"spec":{"template":{"spec":{"securityContext":{"fsGroup":6000}}}}}'
        ~~~
    3. Scale the deployment to 1:

        ~~~sh
        oc -n ${NAMESPACE} scale deployment reversewords-app-storage --replicas 1
        ~~~
9. Let's discover what changed this time:

    1. The directory is now owned by the new FSGroup:

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- ls -ld /mnt/
        ~~~

        ~~~
        drwxrwsrwt. 3 root 6000 133 Feb 12 11:25 /mnt/
        ~~~
    2. Same happens for files that already existed:

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- ls -lrt /mnt/
        ~~~

        ~~~
        total 199008
        -rw-rw-r--. 1 root 6000     13802 Jan 20 10:54 df-output
        -rw-rw-r--. 1 root 6000 203764702 Jan 20 10:55 kubelet
        drwxrws---. 3 root 6000        17 Feb  9 12:33 systemd-private-362be34de0c3408684990c1a68f76806-chronyd.service-iNYlCW
        -rw-rw-r--. 1 1024 6000         0 Feb 12 11:25 testfile
        ~~~

**SupplementalGroups**

`SupplementalGroups` can be used for block and shared, but it is usually used by the later. No chown will be done here. Permissions will be managed at the shared file system level. 

In this demo we will mount a pv inside a pod and we will see how the different configurations for `SupplementalGroup` affect to the files created on the storage.

1. The SCC that we have been using so far, has the following configuration for `SupplementalGroups`:

    > **NOTE**: Below configuration will allow our pods to use any GID on their `supplementalGroups` configuration:

    ~~~yaml
    supplementalGroups:
      type: RunAsAny
    ~~~
2. Scale the deployment to 0:

        ~~~sh
        oc -n ${NAMESPACE} scale deployment reversewords-app-storage --replicas 0
        ~~~

3. We are going to re-use the deployment from the previous lab, let's patch it to clean the `FSGroup` configuration:

    ~~~sh
    oc -n ${NAMESPACE} patch deployment reversewords-app-storage -p '{"spec":{"template":{"spec":{"securityContext":null}}}}'
    ~~~
4. Now we can add the `supplementalGroup` config:

    ~~~sh
    oc -n ${NAMESPACE} patch deployment reversewords-app-storage -p '{"spec":{"template":{"spec":{"securityContext":{"supplementalGroups":[3000]}}}}}'
    ~~~

5. Scale the deployment to 1:

   ~~~sh
   oc -n ${NAMESPACE} scale deployment reversewords-app-storage --replicas 1
   ~~~

5. If we check the group assigned to our pod we will see how it got assigned the secondary group `3000`:

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- id
    ~~~

    > **NOTE**: Group 5000 is still being assigned because the SCC force our pods to run with a group from the 5000-6000 range.

    ~~~sh
    uid=1024(1024) gid=0(root) groups=0(root),3000,5000
    ~~~
    
5. If we check the storage we mounted we will see a couple things:

    1. The directory is owned by the FSgroup id (`5000`) instead of the SupplementalGroup and still has the `SETGID` bit enabled.

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- ls -ld /mnt/
        ~~~

        ~~~
        drwxrwsrwt. 3 root 5000 133 Feb 12 11:25 /mnt/
        ~~~

    2. When we create a file it gets created with our UID and the GID from the folder (remember the SETGID):

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- touch /mnt/testfile2
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- ls -lrt /mnt/
        ~~~

        ~~~
        total 199008
        -rw-rw-r--. 1 root 5000     13802 Jan 20 10:54 df-output
        -rw-rw-r--. 1 root 5000 203764702 Jan 20 10:55 kubelet
        drwxrws---. 3 root 5000        17 Feb  9 12:33 systemd-private-362be34de0c3408684990c1a68f76806-chronyd.service-iNYlCW
        -rw-rw-r--. 1 1024 5000         0 Feb 12 11:25 testfile
        -rw-r--r--. 1 1024 5000         0 Feb 12 12:00 testfile2
        ~~~
6. If this volume was a shared storage being mounted by multiple pods, the supplementalGroup will be set to the owner of the shared storage, then all pods will write with their own UIDs but still have access to files created by other pods.

## **Hands-on Demo 3**

In this demo we will see how we can use `seLinuxContext` strategies. Basically it controls which SELinux context is used by the containers.

Note that all predefined SCCs, except for the privileged SCC, set the seLinuxContext to MustRunAs. This forces pods to use MCS labels, which can be defined in the pod definition, the container image, or provided as a default. If the seLinuxContext strategy is set to MustRunAs and the pod (or image) does not define a label, OpenShift Container Platform defaults to a label chosen from the SCC itself or from the project. In the case of my namespace or project, it defaults to:

~~~yaml
oc get ns $NAMESPACE -o yaml | grep "sa.scc.mcs"
openshift.io/sa.scc.mcs: s0:c27,c9
~~~

1. Now, let's create a deployment called selinux-app without configuring any `SeLinuxContext` and use our restricted-runasuser SCC, where `SELinuxContext` is set to MustRunAs:

> **NOTE**: The application that we are deploying is composed by two containers. Each containers run a netcat binary listening in a different port. This is important, since as they are sharing the pod, they cannot listen in the same port.


~~~yaml
cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: selinux-app
  name: selinux-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: selinux-app
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: selinux-app
    spec:
      serviceAccountName: testuser
      containers:
      - image: quay.io/alosadag/nc
        name: nc1
        env:
        - name: PORT
          value: "8081"
        resources: {} 
      - image: quay.io/alosadag/nc
        name: nc2
        env:
        - name: PORT
          value: "8082"
        resources: {}
status: {}
EOF
~~~

2. Verify that the pod is running and both containers are deployed by checking the READY field equals to 2/2:

~~~sh
oc get pods
NAME                           READY   STATUS    RESTARTS   AGE
selinux-app-7bc9dc5777-9krwj   2/2     Running   0          15s
~~~

3. Check that the MCS assigned to the pod is the default one assigned to the namespace where the application is running:

~~~sh
oc get pod -o yaml selinux-app-7bc9dc5777-9krwj| grep -A1 -i "seLinuxOptions:"
    seLinuxOptions:
      level: s0:c27,c9
~~~

The SELinux context assigned to the parent pid of each container is also the same one:

~~~sh
oc rsh -c nc1 deployment/selinux-app ps -auxZq 1 
LABEL                           USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
system_u:system_r:container_t:s0:c9,c27 3000   1  0.0  0.0  24724  2440 ?        Ss   13:08   0:00 /usr/bin/nc -v -klp 8081

oc rsh -c nc2  deployment/selinux-app ps -auxZq 1 
LABEL                           USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
system_u:system_r:container_t:s0:c9,c27 3000   1  0.0  0.0  24724  2440 ?        Ss   13:08   0:00 /usr/bin/nc -v -klp 8082
~~~

This can be verified at the node level as well. See that the pod label can be seen either from the container or from the host where is running. First, find the node where the pod is running and then connect and check the SELinux context assigned:

~~~sh
oc get pods -o wide
NAME                           READY   STATUS    RESTARTS   AGE    IP             NODE                                      NOMINATED NODE   READINESS GATES
selinux-app-7bc9dc5777-9krwj   2/2     Running   0          5m6s   10.134.2.131   cnfdb2.clus2.t5g.lab.eng.bos.redhat.com   <none>           <none>

oc debug node/cnfdb2.clus2.t5g.lab.eng.bos.redhat.com
Creating debug namespace/openshift-debug-node-m9dxb ...
Starting pod/cnfdb2clus2t5glabengbosredhatcom-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.19.16.3
If you don't see a command prompt, try pressing enter.
sh-4.4# chroot /host

sh-4.4# ps axufZ | grep "/usr/bin/nc -v -klp 808"
system_u:system_r:container_t:s0:c9,c27 3000 3074874 0.0  0.0 24724 2440 ?       Ss   13:08   0:00  \_ /usr/bin/nc -v -klp 8081
system_u:system_r:container_t:s0:c9,c27 3000 3074967 0.0  0.0 24724 2436 ?       Ss   13:08   0:00  \_ /usr/bin/nc -v -klp 8082
~~~

4. Ok, so now let's imagine we have a security requirement to have both containers as much isolated one from each other. We can assign a different SELinux MCS to each one. So, let's scale to 0 the deployment, remove the restricted-runasuser SCC and create a new one, which will be similar to the default restricted profile but maintaining the priority to 1 so it is selected instead of the restricted SCC. 

~~~sh
oc scale deploy selinux-app --replicas=0
~~~

~~sh
oc delete scc restricted-runasuser
securitycontextconstraints.security.openshift.io "restricted-runasuser" deleted
~~~

~~~yaml
cat <<EOF | oc -n ${NAMESPACE} create -f -
kind: SecurityContextConstraints
metadata:
  name: restricted-runasuser
priority: 1
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

5. Let's patch the deployment by adding to the container named nc1 an specific SELinux label (MCS) different from the one assigned to the namespace:

~~~sh
oc -n ${NAMESPACE} patch deployment selinux-app -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"nc1"}],"containers":[{"name":"nc1","securityContext":{"seLinuxOptions":{"level":"s0:c123,c456"}}}]}}}}'
~~~

6. Now, scale it up:

~~~sh
oc scale deploy selinux-app --replicas=1
~~~


7. See that the deployment is not able to create the pod since the seLinuxOptions.level is invalid because the SCC does not allow us to select any other:

~~~sh
0s          Warning   FailedCreate        replicaset/selinux-app-6575fb4fff                Error creating: pods "selinux-app-6575fb4fff-" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.seLinuxOptions.level: Invalid value: "s0:c123,c456": must be s0:c27,c9 spec.containers[0].securityContext.seLinuxOptions.level: Invalid value: "s0:c123,c456": must be s0:c27,c9]
~~~

8. Now, let's patch the restricted-runasuser SCC by setting the `SeLinuxContext` to **RunAsAny**. Remember that if seLinuxContext is set to RunAsAny, then no default labels are provided, and the container determines the final label if it is not set explicitly. In our case, we are going to force the label for nc1 and let nc2 container to use a default one assigned by the runtime.


~~~sh
oc patch scc restricted-runasuser -p '{"seLinuxContext":{"type":"RunAsAny"}}' --type merge
~~~

Then, after a couple of minutes, verify that the pod is running successfully with the proper SELinux contexts:

~~~sh
oc rsh -c nc1 deployment/selinux-app ps -auxZq 1 
LABEL                           USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
system_u:system_r:container_t:s0:c123,c456 1000720+ 1 0.0  0.0 24724 2396 ?      Ss   13:31   0:00 /usr/bin/nc -v -klp 8081

oc rsh -c nc2 deployment/selinux-app ps -auxZq 1 
LABEL                           USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
system_u:system_r:container_t:s0:c478,c821 1000720+ 1 0.0  0.0 24724 2388 ?      Ss   13:31   0:00 /usr/bin/nc -v -klp 8082
~~~

