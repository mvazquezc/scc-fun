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

    ~~~sh
    cat <<EOF | oc create -f -
    kind: SecurityContextConstraints
    metadata:
      name: restricted-runasuser
    priority: 2
    readOnlyRootFilesystem: false
    requiredDropCapabilities:
    - ALL
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

    ~~~sh
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

    > **NOTE**: Below output might be different on your cluster depending on the range assigned to your namespace.

    ~~~sh
    1000650000
    ~~~

9. We can now patch the deployment, so we explicitly request a UID from the range:

    > **NOTE**: You might need to change the UID in the command below to use a UID from your range.

    ~~~sh
    oc -n ${NAMESPACE} patch deployment reversewords-app -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"reversewords"}],"containers":[{"name":"reversewords","securityContext":{"runAsUser":1000650015}}]}}}}'
    ~~~

10. Now the pod will be running with the requested UID `1000650015`:

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app -- whoami
    ~~~

    ~~~sh
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

    ~~~sh
    status:
    <OUTPUT_OMITTED>
      - lastTransitionTime: "2023-10-16T16:51:57Z"
        lastUpdateTime: "2023-10-16T16:51:57Z"
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

    ~~~sh
    2000
    ~~~

14. One interesting experiment is what will happen if instead of the range [2000,2500] we force the container to use an UID from the namespace range:

    ~~~sh
    # Get a UID from the namespace range
    CONTAINER_UID=$(oc get ns ${NAMESPACE} -o jsonpath='{.metadata.annotations.openshift\.io\/sa\.scc\.uid-range}' | awk -F "/" '{print $1}')
    # Patch the deployment with the new UID
    oc -n ${NAMESPACE} patch deployment reversewords-app -p "{\"spec\":{\"template\":{\"spec\":{\"$setElementOrder/containers\":[{\"name\":\"reversewords\"}],\"containers\":[{\"name\":\"reversewords\",\"securityContext\":{\"runAsUser\":${CONTAINER_UID}}}]}}}}"
    ~~~

    1. We can think from the results that it is possible then to specify any of the ranges defined either on the SCC or the namespace:

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app -- whoami
        ~~~

        > **NOTE**: Below output might be different on your cluster depending on the range assigned to your namespace.

        ~~~sh
        1000650010
        ~~~

    2. However, if we pay attention we can see that our SA can use two SCCs: the custom one which has higher priority and the default one: restricted-v2. In this case, since the custom one defines a different range than the requested, it is skipped. The restricted-v2 one matches perfectly the user request, so it is used:

        ~~~sh
        oc -n ${NAMESPACE} get pod -l app=reversewords-app -o 'custom-columns=NAME:metadata.name,APPLIED SCC:metadata.annotations.openshift\.io/scc'
        ~~~

        ~~~sh
        NAME                                APPLIED SCC
        reversewords-app-7d8558f8f4-hhx6v   restricted-v2
        ~~~

> **NOTE:** You can check the SCC or SCCs that can be used to fulfill the pod spec by executing the scc-review command. The result is a list of SCC ordered.

**MustRunAs Strategy**

1. We can also force all our pods to run with a specific uid, let's patch our SCC first:

    ~~~sh
    oc patch scc restricted-runasuser -p '{"runAsUser":{"type":"MustRunAs","uid":1024,"uidRangeMax":null,"uidRangeMin":null}}' --type merge
    ~~~

2. Let's patch the deployment so we don't request any uid:

    ~~~sh
    oc -n ${NAMESPACE} patch deployment reversewords-app -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"reversewords"}],"containers":[{"name":"reversewords","securityContext":null}]}}}}'
    ~~~

3. If we check the UID now, we will see that it's the one we forced `1024`:

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app -- whoami
    ~~~

    ~~~sh
    1024
    ~~~

4. If we try to patch the deployment to make it run with a different UID, it won't work.

**RunAsAny and MustRunAsNonRoot Strategies**

These two strategies can be used as well. `RunAsAny` allows our pods to run with any UID, including `UID 0`, whereas `MustRunAsNonRoot` allows our pods to run with any UID, but `UID 0`.

These strategies do not provide default values.

**runAsGroup**

By default (unless you specify runAsGroup) the user running the container is always a member of the root group (GID 0), which doesn't grant any privileges but allows the user to read/write files accessible by GID 0. That's required for some apps that need to read config files owned by root. We can think of an example nginx pod which has its configuration files owned by root with '0640' permission. The nginx process could be run with a random UID (which cannot read the root owned files), but since the user is a member of the root group it will be able to read the config files and run nginx properly.

## **Hands-on Demo 2**

In this demo we will see how we can use `FSGroup` and `SupplementalGroups` strategies for accessing storage from our pods.

Both strategies have the same config modes we already saw for `RunAsUser`, so instead of focusing on how the different config modes affect the user that runs the pod, we will focus on what are the differences between using `FSGroup` and `SupplementalGroups` and how they impact the storage mounted on the pod.

**FSGroup**

As you might know, `FSGroup` is mainly used for block storage because it will chown the content of the storage so the group id defined can read/write content. As you can imagine, running such operation in a shared storage is not a good idea.

We can still decide how the content is chowned using the `FSGroupChangePolicy`, but still, there are storage plugins like nfs that won't take into account `FSGroup` strategies.

* `OnRootMismatch`: Only change permissions and ownership if permission and ownership of root directory does not match with expected permissions of the volume. This could help shorten the time it takes to change ownership and permission of a volume.

* `Always`: Always change permission and ownership of the volume when volume is mounted.

In this demo we will mount a pv inside a pod, and we will see how the different configurations for `FSGroup` affect to the files created on the storage.

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

2. We are going to create a block storage volume using the `local` plugin in a `PersistentVolume` (PV) which will give us access to a folder inside the `worker` nodes:

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

    > **NOTE**: Below patch will allow pods using our SCC to use GIDs between 6000 and 7000. If no `FSGroup` is set in the `securityContext` configuration, it will get the first GID from the range, `5000` in this case.

    ~~~sh
    oc patch scc restricted-runasuser -p '{"fsGroup":{"ranges":[{"max":7000,"min":6000}]}}' --type merge
    ~~~

5. Now, it's time to create the deployment, we will use `FSGroup` 6005:

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
            fsGroup: 6005
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

6. If we check the group assigned to our pod we will see how it got assigned the secondary group `6005`:

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- id
    ~~~

    ~~~sh
    uid=1024(1024) gid=0(root) groups=0(root),6005
    ~~~

7. If we check the storage we mounted we will see a couple of things:

    1. The directory is owned by the configured FSGroup and has the `SETGID` bit enabled.

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- ls -ld /mnt/
        ~~~

        ~~~sh
        drwxrwsrwt. 3 root 6005 117 Oct 16 16:55 /mnt/
        ~~~

    2. When we create a file it gets created with our UID and GID:

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- touch /mnt/testfile
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- ls -lrt /mnt/
        ~~~

        ~~~sh
        total 0
        drwxrws---. 3 root 6005 17 Oct 16 14:59 systemd-private-5b22965e6b564f98b7705a6915a3416d-chronyd.service-Ti8KQF
        -rw-r--r--. 1 1024 6005  0 Oct 16 16:55 testfile
        ~~~

8. We mentioned that `FSGroup` chowns the content in the volume, let's try to change the group used to run our pod:

    1. Scale the deployment to 0:

        ~~~sh
        oc -n ${NAMESPACE} scale deployment reversewords-app-storage --replicas 0
        ~~~

    2. Patch the deployment so it uses the new group, `7000` this time:

        ~~~sh
        oc -n ${NAMESPACE} patch deployment reversewords-app-storage -p '{"spec":{"template":{"spec":{"securityContext":{"fsGroup":7000}}}}}'
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

        ~~~sh
        drwxrwsrwt. 5 root 7000 4096 Oct 16 16:55 /mnt/
        ~~~

    2. The same happens for files that already existed:

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- ls -lrt /mnt/
        ~~~

        ~~~sh
        total 0
        drwxrws---. 3 root 7000 17 Oct 16 14:59 systemd-private-5b22965e6b564f98b7705a6915a3416d-chronyd.service-Ti8KQF
        -rw-rw-r--. 1 1024 7000  0 Oct 16 16:55 testfile
        ~~~

**SupplementalGroups**

`SupplementalGroups` can be used for block and shared, but it is usually used by the later. No chown will be done here. Permissions will be managed at the shared file system level. 

In this demo we will mount a PV inside a pod, and we will see how the different configurations for `SupplementalGroup` affect to the files created on the storage.

1. In order to have SharedStorage we will deploy a NFS server:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create -f -
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: nfs-server
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: nfs-server-anyuid-scc
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: system:openshift:scc:anyuid
    subjects:
    - kind: ServiceAccount
      name: nfs-server
      namespace: ${NAMESPACE}
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        app: nfs-server
      name: nfs-server
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nfs-server
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: nfs-server
        spec:
          serviceAccountName: nfs-server
          containers:
          - image: quay.io/mavazque/nfs-server:latest
            name: nfs-server
            securityContext:
              runAsUser: 0
            ports:
            - containerPort: 2049
            resources: {}
    status: {}
    ---
    apiVersion: v1
    kind: Service
    metadata:
      creationTimestamp: null
      labels:
        app: nfs-server
      name: nfs-server
    spec:
      ports:
      - name: 2049-2049
        port: 2049
        protocol: TCP
        targetPort: 2049
      selector:
        app: nfs-server
      type: ClusterIP
    status:
      loadBalancer: {}
    EOF
    ~~~

2. Create a PV and a PVC to be used by the app:

    ~~~sh
    # Get ClusterIP Service IP
    NFS_SERVER=$(oc -n ${NAMESPACE} get svc nfs-server -o jsonpath='{.spec.clusterIP}')
    # Objects creation
    cat <<EOF | oc -n ${NAMESPACE} create -f -
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: nfs-shared-test-volume
    spec:
      capacity:
        storage: 500Mi
      accessModes:
        - ReadWriteMany
      nfs:
        server: ${NFS_SERVER}
        path: "/nfs-share"
      mountOptions:
        - port=2049
        - mountport=2049
        - nfsvers=3
        - tcp
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nfs-shared-test-volume-claim
    spec:
      accessModes:
        - ReadWriteMany
      storageClassName: ""
      volumeName: nfs-shared-test-volume
      resources:
        requests:
          storage: 500Mi
    EOF
    ~~~

3. The SCC that we have been using so far, has the following configuration for `SupplementalGroups`:

    > **NOTE**: Below configuration will allow our pods to use any GID on their `supplementalGroups` configuration:

    ~~~yaml
    supplementalGroups:
      type: RunAsAny
    ~~~

4. Now, it's time to create the deployment, we will use `supplementalGroups` 5000 which is the one used in our NFS share:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        app: reversewords-app-shared-storage
      name: reversewords-app-shared-storage
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: reversewords-app-shared-storage
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: reversewords-app-shared-storage
        spec:
          serviceAccountName: testuser
          volumes:
          - name: test-volume
            persistentVolumeClaim:
              claimName: nfs-shared-test-volume-claim
          securityContext:
            supplementalGroups: [5000]
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

5. If we check the group assigned to our pod we will see how it got assigned the secondary group `5000`:

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- id
    ~~~

    > **NOTE**: Group 6000 is still being assigned because the SCC force our pods to run with a group from the 6000-7000 range.

    ~~~sh
    uid=1024(1024) gid=0(root) groups=0(root),5000,6000
    ~~~

5. If we check the storage we mounted we will see a couple things:

    1. This time the directory is not owned by the FSGroup (`6000`), instead it's owned by the GID configured in the shared storage (`5000`) in this case. It has the `SETGID` enabled because the NFS share is exported with it.

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- ls -ld /mnt/
        ~~~

        ~~~sh
        drwxrwsr-x. 1 5000 5000 26 Feb 16  2021 /mnt/
        ~~~

    2. When we create a file it gets created with our UID and the GID from the folder (remember the SETGID):

        ~~~sh
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- touch /mnt/testfile2
        oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- ls -lrt /mnt/
        ~~~

        ~~~sh
        total 4
        -rw-rw----. 1 5000 5000 38 Feb 16  2021 testfile.txt
        -rw-r--r--. 1 1024 5000  0 Oct 16 16:59 testfile2
        ~~~

6. If this volume was a shared storage being mounted by multiple pods, since the supplementalGroup will be set to the owner of the shared storage, then all pods will write with their own UIDs but still have access to files created by other pods.

## **Hands-on Demo 3**

In this demo we will see how we can use `seLinuxContext` strategies. Basically it controls which SELinux context is used by the containers.

Note that all predefined SCCs, except for the privileged SCC, set the seLinuxContext to MustRunAs. This forces pods to use MCS labels, which can be defined in the securityContext at the pod level, at the container level, or provided as a default. If the `seLinuxContext` strategy is set to `MustRunAs` and the pod (or image) does not define a label, OpenShift Container Platform defaults to a label chosen from the SCC itself or from the namespace. In the case of my namespace, it defaults to:

~~~sh
oc get ns ${NAMESPACE} -o yaml | grep "sa.scc.mcs"
~~~

> **NOTE**: Below output might be different on your cluster depending on the range assigned to your namespace.

~~~sh
openshift.io/sa.scc.mcs: s0:c27,c9
~~~

1. Now, let's create a deployment called selinux-app without configuring any `SeLinuxContext` and which uses our restricted-runasuser SCC, where `SELinuxContext` is set to MustRunAs.

    ~~~sh
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
          - image: quay.io/mavazque/trbsht:latest
            name: nc1
            resources: {} 
          - image: quay.io/mavazque/trbsht:latest
            name: nc2
            resources: {}
    status: {}
    EOF
    ~~~

2. Verify that the pod is running and both containers are deployed by checking the READY field equals to 2/2:

    ~~~sh
    oc -n ${NAMESPACE} get pods -l app=selinux-app
    ~~~

    ~~~sh
    NAME                           READY   STATUS    RESTARTS   AGE
    selinux-app-6996d479cb-vwtvv   2/2     Running   0          8s
    ~~~

3. Check that the MCS assigned to the pod is the default one assigned to the namespace where the application is running:

    ~~~sh
    oc -n ${NAMESPACE} get pod -l app=selinux-app -o 'custom-columns=NAME:metadata.name,APPLIED MCS:spec.securityContext.seLinuxOptions.level'
    ~~~

    ~~~sh
    NAME                                APPLIED MCS
    selinux-app-6996d479cb-vwtvv        s0:c27,c14
    ~~~

    1. The SELinux context assigned to the parent pid of each container is also the same one:

        ~~~sh
        oc -n ${NAMESPACE} exec -c nc1 deployment/selinux-app -- ps -auxZq 1
        ~~~

        ~~~sh
        LABEL                           USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
        system_u:system_r:container_t:s0:c14,c27 1024  1  0.0  0.0   2468   900 ?        Ss   17:03   0:00 sleep infinity
        ~~~

        ~~~sh
        oc -n ${NAMESPACE} exec -c nc2 deployment/selinux-app -- ps -auxZq 1 
        ~~~

        ~~~sh
        LABEL                           USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
        system_u:system_r:container_t:s0:c14,c27 1024  1  0.0  0.0   2468   960 ?        Ss   17:03   0:00 sleep infinity
        ~~~

    2. This can be verified at the node level as well. See that the pod label can be seen either from the container or from the host where it is running. First, find the node where the pod is running and then connect and check the SELinux context assigned:

        ~~~sh
        # Get the node that executed the pod
        POD_NODE=$(oc -n ${NAMESPACE} get pod -l app=selinux-app -o jsonpath='{.items[*].spec.nodeName}')
        # Connected to the node
        oc debug node/${POD_NODE}
        # Chroot and grep for the NC processes
        chroot /host
        ps axufZ | grep "c14,c27" | grep "sleep"
        ~~~

        ~~~sh
        system_u:system_r:container_t:s0:c14,c27 1024 176415 0.0  0.0 2468  900 ?        Ss   17:03   0:00  \_ sleep infinity
        system_u:system_r:container_t:s0:c14,c27 1024 176471 0.0  0.0 2468  960 ?        Ss   17:03   0:00  \_ sleep infinity
        ~~~

4. Ok, so now let's imagine we have a security requirement to have both containers as much isolated as possible from each other. We can assign a different SELinux MCS to each one.

    1. Let's scale the deployment to 0 and patch it so one of the containers uses a different MCS label from the one assigned to the namespace:

        ~~~sh
        oc -n ${NAMESPACE} scale deployment selinux-app --replicas 0
        ~~~

        ~~~sh
        oc -n ${NAMESPACE} patch deployment selinux-app -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"nc1"}],"containers":[{"name":"nc1","securityContext":{"seLinuxOptions":{"level":"s0:c123,c456"}}}]}}}}'
        ~~~

    2. We can scale the deployment back to 1:

        ~~~sh
        oc -n ${NAMESPACE} scale deployment selinux-app --replicas 1
        ~~~

    3. See that the deployment is not able to create the pod since the seLinuxOptions.level is invalid because the SCC does not allow us to select any other:

        ~~~sh
        oc -n ${NAMESPACE} get deployment selinux-app -o yaml | grep -A20 ^status:
        ~~~

        ~~~sh
        status:
          conditions:
          <OUTPUT_OMITTED>
            - lastTransitionTime: "2023-10-16T17:07:22Z"
              lastUpdateTime: "2023-10-16T17:07:22Z"
              message: 'pods "selinux-app-8947dfdbc-" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.seLinuxOptions.level: Invalid value: "s0:c123,c456": must be s0:c27,c14 spec.containers[0].securityContext.seLinuxOptions.level: Invalid value: "s0:c123,c456": must be s0:c27,c14]'
        <OUTPUT_OMITTED>
        ~~~

    4. Now, let's patch the restricted-runasuser SCC by setting the `SeLinuxContext` to **RunAsAny**. Remember that if seLinuxContext is set to RunAsAny, then no default labels are provided, and the container determines the final label if it is not set explicitly. In our case, we are going to force the label for nc1 and let nc2 container to use a default one assigned by the runtime.

        ~~~sh
        oc patch scc restricted-runasuser -p '{"seLinuxContext":{"type":"RunAsAny"}}' --type merge
        ~~~

5. After a couple of minutes, verify that the pod is running successfully with the proper SELinux contexts:

    ~~~sh
    oc -n ${NAMESPACE} exec -c nc1 deployment/selinux-app -- ps -auxZq 1
    ~~~

    ~~~sh
    LABEL                           USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    system_u:system_r:container_t:s0:c123,c456 1024 1 0.0  0.0   2468   992 ?        Ss   17:08   0:00 sleep infinity
    ~~~

    ~~~sh
    oc -n ${NAMESPACE} exec -c nc2 deployment/selinux-app -- ps -auxZq 1 
    ~~~

    ~~~sh
    LABEL                           USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    system_u:system_r:container_t:s0:c533,c936 1024 1 0.0  0.0   2468   936 ?        Ss   17:08   0:00 sleep infinity
    ~~~