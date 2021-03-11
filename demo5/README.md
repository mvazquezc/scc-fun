# **Debugging SCCs Issues**

In this demo we will learn how we can debug issues related to SCCs in our cluster.

## **Hands-on Demo 1**

We have a workload that requires binding to port 80. We create the deployment but pods keep failing.

1. Create a new namespace for running our tests:

    ~~~sh
    NAMESPACE=debug-sccs
    oc create ns ${NAMESPACE}
    ~~~
2. Create the following deployment

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create -f -
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
          serviceAccountName: default
          containers:
          - image: quay.io/mavazque/reversewords:ubi8
            name: reversewords
            resources: {}
            env:
            - name: APP_PORT
              value: "80"
    status: {}
    EOF
    ~~~
3. Debug why pod is failing and fix the issue using an SCC from the ones included in OpenShift out of the box.

**Solution**

<details>
  <summary>Click here to see solution</summary>

  1. Check the pod logs

      ~~~sh
      oc -n ${NAMESPACE} logs -l app=reversewords-app
      ~~~

      ~~~
      2021/02/08 10:58:34 Starting Reverse Api v0.0.17 Release: NotSet
      2021/02/08 10:58:34 Listening on port 80
      2021/02/08 10:58:34 listen tcp :80: bind: permission denied
      ~~~
  2. From the logs we can see that the pod doesn't have permissions to bind to port 80.
  3. We could bind to that port if we were running as UID 0.
  4. Create a ServiceAccount for running our deployment workloads:
  
      ~~~sh
      oc -n ${NAMESPACE} create serviceaccount reversewordsapp
      ~~~
  5. Assign the SCC `anyuid` to the SA:

      ~~~sh
      oc -n ${NAMESPACE} adm policy add-scc-to-user anyuid -z reversewordsapp
      ~~~
  6. Patch the deployment so it uses the new SA we created:

      ~~~sh
      oc -n ${NAMESPACE} patch deployment reversewords-app -p '{"spec":{"template":{"spec":{"serviceAccountName":"reversewordsapp"}}}}' --type merge
      ~~~
  7. Patch the deployment so container `reversewords` runs with UID 0:

      ~~~sh
      oc -n ${NAMESPACE} patch deployment reversewords-app -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"reversewords"}],"containers":[{"name":"reversewords","securityContext":{"runAsUser":0}}]}}}}'
      ~~~
  8. Check pod logs:

      ~~~sh
      oc -n ${NAMESPACE} logs -l app=reversewords-app
      ~~~

      ~~~
      2021/02/08 11:03:21 Starting Reverse Api v0.0.17 Release: NotSet
      2021/02/08 11:03:21 Listening on port 80
      ~~~
</details>

## **Hands-on Demo 2**

We have a workload that requires mounting a hostpath. We create the deployment but pods are not being created.

1. Set the namespace for running our tests:

    ~~~sh
    NAMESPACE=debug-sccs
    ~~~
2. Create the following deployment

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        app: reversewords-app-hostpath
      name: reversewords-app-hostpath
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: reversewords-app-hostpath
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: reversewords-app-hostpath
        spec:
          serviceAccountName: default
          containers:
          - image: quay.io/mavazque/reversewords:ubi8
            name: reversewords
            resources: {}
            volumeMounts:
            - mountPath: /test-mount/test.txt
              name: hostpath-volume
          volumes:
          - name: hostpath-volume
            hostPath:
              path: /tmp/test.txt
              type: FileOrCreate
    status: {}
    EOF
    ~~~
3. Debug why pod is not being created and fix the issue using a custom SCC.

**Solution**

<details>
  <summary>Click here to see solution</summary>

  1. Check deployment status

      ~~~sh
      oc -n ${NAMESPACE} get deployment reversewords-app-hostpath -o yaml | grep -A100  ^status:
      ~~~

      ~~~
      status:
        conditions:
        - lastTransitionTime: "2021-02-08T11:24:05Z"
          lastUpdateTime: "2021-02-08T11:24:05Z"
          message: Created new replica set "reversewords-app-hostpath-598657994b"
          reason: NewReplicaSetCreated
          status: "True"
          type: Progressing
        - lastTransitionTime: "2021-02-08T11:24:05Z"
          lastUpdateTime: "2021-02-08T11:24:05Z"
          message: Deployment does not have minimum availability.
          reason: MinimumReplicasUnavailable
          status: "False"
          type: Available
        - lastTransitionTime: "2021-02-08T11:24:05Z"
          lastUpdateTime: "2021-02-08T11:24:05Z"
          message: 'pods "reversewords-app-hostpath-598657994b-" is forbidden: unable to validate against any security context constraint: [spec.volumes[0]: Invalid value: "hostPath": hostPath volumes are not allowed to be used]'
          reason: FailedCreate
          status: "True"
          type: ReplicaFailure
        observedGeneration: 1
        unavailableReplicas: 1
      ~~~
  2. From the status we can see that the pod cannot be validated against any SCC which allow hostPath mounts.
  3. We could mount that path if we had access to an SCC which allows this kind of mounts.
  4. Create a ServiceAccount for running our deployment workloads:
  
      ~~~sh
      oc -n ${NAMESPACE} create serviceaccount reversewordsapp-hostpath
      ~~~
  5. Create a SCC based on the restricted one but with permissions to mount `hostPath` volumes:

      > **NOTE**: We added `hostPath` to the list of allowed volumes. And set `allowHostDirVolumePlugin` to `true`.

      ~~~sh
      cat <<EOF | oc create -f -
      kind: SecurityContextConstraints
      metadata:
        name: restricted-hostpathmount
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
      - hostPath
      allowHostDirVolumePlugin: true
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
  5. Assign the SCC `restricted-hostpathmount` to the SA:

      ~~~sh
      oc -n ${NAMESPACE} adm policy add-scc-to-user restricted-hostpathmount -z reversewordsapp-hostpath
      ~~~
  6. Patch the deployment so it uses the new SA we created:

      ~~~sh
      oc -n ${NAMESPACE} patch deployment reversewords-app-hostpath -p '{"spec":{"template":{"spec":{"serviceAccountName":"reversewordsapp-hostpath"}}}}' --type merge
      ~~~
  7. Scale the deployment to 0 and back to 1 so it gets the latest configuration:

      ~~~sh
      oc -n ${NAMESPACE} scale deployment reversewords-app-hostpath --replicas=0
      oc -n ${NAMESPACE} scale deployment reversewords-app-hostpath --replicas=1
      ~~~
  8. Check pod logs:

      ~~~sh
      oc -n ${NAMESPACE} logs -l app=reversewords-app-hostpath
      ~~~

      ~~~
      2021/02/08 11:34:21 Starting Reverse Api v0.0.17 Release: NotSet
      2021/02/08 11:34:21 Listening on port 8080
      ~~~
  9. List our hostPath volume:

      ~~~sh
      oc -n ${NAMESPACE} exec deploy/reversewords-app-hostpath -- ls -l /test-mount/test.txt
      ~~~

      ~~~
      -rw-r--r--. 1 root root 0 Feb  8 11:34 /test-mount/test.txt
      ~~~
</details>

## **Hands-on Demo 3**

We have a workload that requires running with a given UID (1024). We created the deployment but pods are not being created. 

1. Set the namespace for running our tests:

    ~~~sh
    NAMESPACE=debug-sccs
    ~~~
2. Create the following deployment

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        app: reversewords-app-uid
      name: reversewords-app-uid
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: reversewords-app-uid
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: reversewords-app-uid
        spec:
          serviceAccountName: default
          containers:
          - image: quay.io/mavazque/reversewords:ubi8
            name: reversewords
            resources: {}
          securityContext:
            runAsUser: 1024
    status: {}
    EOF
    ~~~
3. Debug why pod is not being created and fix the issue using an SCC from the ones included in OpenShift out of the box.

**Solution**

<details>
  <summary>Click here to see solution</summary>
  
  1. Check deployment status

      ~~~sh
      oc -n ${NAMESPACE} get deployment reversewords-app-uid -o yaml | grep -A100  ^status:
      ~~~

      ~~~
      status:
        conditions:
        - lastTransitionTime: "2021-02-08T11:56:35Z"
          lastUpdateTime: "2021-02-08T11:56:35Z"
          message: Created new replica set "reversewords-app-uid-7b9c7c7f59"
          reason: NewReplicaSetCreated
          status: "True"
          type: Progressing
        - lastTransitionTime: "2021-02-08T11:56:35Z"
          lastUpdateTime: "2021-02-08T11:56:35Z"
          message: Deployment does not have minimum availability.
          reason: MinimumReplicasUnavailable
          status: "False"
          type: Available
        - lastTransitionTime: "2021-02-08T11:56:35Z"
          lastUpdateTime: "2021-02-08T11:56:35Z"
          message: 'pods "reversewords-app-uid-7b9c7c7f59-" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.runAsUser: Invalid value: 1024: must be in the ranges: [1000610000, 1000619999]]'
          reason: FailedCreate
          status: "True"
          type: ReplicaFailure
        observedGeneration: 1
        unavailableReplicas: 1
      ~~~
  2. From the status we can see that the pod cannot be validated against any SCC which allow running with an arbitrary UID.
  3. Create a ServiceAccount for running our deployment workloads:
  
      ~~~sh
      oc -n ${NAMESPACE} create serviceaccount reversewordsapp-uid
      ~~~
  4. We have a couple SCCs that can make this work `anyuid` and `nonroot`. Since we don't need to run with UID 0, `nonroot` is a better choice.
  5. Assign the SCC `nonroot` to the SA:

      ~~~sh
      oc -n ${NAMESPACE} adm policy add-scc-to-user nonroot -z reversewordsapp-uid
      ~~~
  6. Patch the deployment so it uses the new SA we created:

      ~~~sh
      oc -n ${NAMESPACE} patch deployment reversewords-app-uid -p '{"spec":{"template":{"spec":{"serviceAccountName":"reversewordsapp-uid"}}}}' --type merge
      ~~~
  7. Check pod logs:

      ~~~sh
      oc -n ${NAMESPACE} logs -l app=reversewords-app-uid
      ~~~

      ~~~
      2021/02/08 12:01:42 Starting Reverse Api v0.0.17 Release: NotSet
      2021/02/08 12:01:42 Listening on port 8080
      ~~~
  9. Check UID assigned to the container:

      ~~~sh
      oc -n ${NAMESPACE} exec deploy/reversewords-app-uid -- whoami
      ~~~

      ~~~
      1024
      ~~~
</details>

## **Hands-on Demo 4**

We have created a custom SCC based on the `restricted` one, we want all pods in the namespace `debug-sccs` using the `custom-scc` SA to run with the `restricted-custom` SCC but for some reason, pods are getting a different SCC assigned.
 
Debug what happens and change the required configurations so we get the `custom-scc` assigned.

1. Set the namespace for running our tests:

    ~~~sh
    NAMESPACE=debug-sccs
    ~~~
2. Create the `restricted-custom` SCC:

    ~~~sh
    cat <<EOF | oc create -f -
    kind: SecurityContextConstraints
    metadata:
      name: restricted-custom
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
3. Create the `custom-scc` SA :

    ~~~sh
    oc -n ${NAMESPACE} create sa custom-scc
    ~~~
4. Give access to `custom-scc` to the `restricted-custom` SCC:

    ~~~sh
    oc -n ${NAMESPACE} adm policy add-scc-to-user restricted-custom system:serviceaccount:${NAMESPACE}:custom-scc
    ~~~

5. Create the following deployment:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        app: reversewords-app-customscc
      name: reversewords-app-customscc
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: reversewords-app-customscc
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: reversewords-app-customscc
        spec:
          serviceAccountName: custom-scc
          containers:
          - image: quay.io/mavazque/reversewords:ubi8
            name: reversewords
            resources: {}
    status: {}
    EOF
    ~~~
6. Check the SCC assigned to the pod:

    ~~~sh
    oc -n ${NAMESPACE} get pod -l app=reversewords-app-customscc -o 'custom-columns=NAME:metadata.name,APPLIED SCC:metadata.annotations.openshift\.io/scc'
    ~~~

    > **NOTE**: `restricted` SCC was applied instead of `restricted-custom`.
    ~~~
    NAME                                         APPLIED SCC
    reversewords-app-customscc-b697b8c5d-w52hn   restricted
    ~~~
7. Debug why pod is not being assigned the desired SCC and fix the issue by modifying the `custom-scc` as needed.

**Solution**

<details>
  <summary>Click here to see solution</summary>
  
  1. Remember that all authenticated users have access to the `restricted` SCC and that the SCCs are ordered as follows:

      1. If the SCCs have different priorities, higher priority first.
      2. If priority is the same, the most restrictive first.
      3. If priority and restriction level are the same, by alphabetical order first.
  2. Our SCCs have the same priority `null` with equals to 0.
  3. Our SCCs have the same restriction level.
  4. Our custom SCC when ordered alphabeticaly will be after the `restricted` one.
  5. We need to increase the priority of the `restricted-custom` SCC so it has more priority and gets 1st on the list:

      ~~~sh
      oc patch scc restricted-custom -p '{"priority":1}' --type merge
      ~~~
  6. Force the app pod to be recreated:

      ~~~sh
      oc -n ${NAMESPACE} delete pod -l app=reversewords-app-customscc
      ~~~
  7. Check the SCC assigned to the new pod:

      ~~~sh
      oc -n ${NAMESPACE} get pod -l app=reversewords-app-customscc -o 'custom-columns=NAME:metadata.name,APPLIED SCC:metadata.annotations.openshift\.io/scc'
      ~~~

      > **NOTE**: This time the `restricted-custom` SCC got prioritized.
      
      ~~~
      NAME                                         APPLIED SCC
      reversewords-app-customscc-b697b8c5d-p2ltv   restricted-custom
      ~~~
</details>

## **Hands-on Demo 5**

We have a NFS server which is providing shared storage. The NFS share is configured so it can be accessed by the group `5000`. We need our application to connect to this share and read/write the content inside the share. For some reason we are not able to read/write the content. 

Debug the issue and fix the required configurations so our application can read/write the content from the share.

1. Create a new namespace for running our tests:

    ~~~sh
    NAMESPACE=debug-scc-shared-storage
    oc create ns ${NAMESPACE}
    ~~~
2. Create the `nfs-server` deployment:

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
3. Create a PV and a PVC to be used by the app for attaching to the NFS Server

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
4. Create the app deployment

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
          volumes:
          - name: test-volume
            persistentVolumeClaim:
              claimName: nfs-shared-test-volume-claim
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
5. Try to read/write to the NFS Share

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- cat /mnt/testfile.txt
    ~~~

    ~~~
    cat: /mnt/testfile.txt: Permission denied
    command terminated with exit code 1
    ~~~

    ~~~sh
    oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- touch /mnt/shared-storage-1
    ~~~

    ~~~
    touch: cannot touch '/mnt/shared-storage-1': Permission denied
    command terminated with exit code 1
    ~~~

**Solution**

<details>
  <summary>Click here to see solution</summary>
  
  1. As said in the lab description, the NFS Share uses the group `5000` as the collaborative group for the NFS Share. That means that we need to assign this group to the user running our container.
  2. Let's review the NFS Share config from the app pod:

      ~~~sh
      oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- ls -ld /mnt
      ~~~

      ~~~
      drwxrwsr-x. 1 5000 5000 26 Feb 16 19:03 /mnt
      ~~~

      ~~~sh
      oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- ls -lrt /mnt/
      ~~~

      ~~~
      total 4
      -rw-rw----. 1 5000 5000 38 Feb 16 18:26 testfile.txt
      ~~~
  3. As we can see, the folder is owned by UID,GID 5000 and so is the file.
  4. Let's check which user is running our container:

      ~~~sh
      oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- id
      ~~~

      ~~~
      uid=1000700000(1000700000) gid=0(root) groups=0(root),1000700000
      ~~~
  5. As you can see the user is not part of the GID 5000, so we need to fix that:

      ~~~sh
      oc -n ${NAMESPACE} patch deployment reversewords-app-shared-storage -p '{"spec":{"template":{"spec":{"securityContext":{"supplementalGroups":[5000]}}}}}'
      ~~~
  6. If we check the user running the container again:

      ~~~sh
      oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- id
      ~~~

      ~~~
      uid=1000700000(1000700000) gid=0(root) groups=0(root),5000,1000700000
      ~~~
  7. Since now we are part of the group we will be able to read and write to the shared storage:

      ~~~sh
      oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- cat /mnt/testfile.txt
      ~~~

      ~~~
      This is a testfile owned by user 5000
      ~~~

      ~~~sh
      oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- touch /mnt/shared-storage-1
      oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-shared-storage -- ls -l /mnt/shared-storage-1
      ~~~

      ~~~
      -rw-r--r--. 1 1000700000 5000 0 Feb 17 11:17 /mnt/shared-storage-1
      ~~~
</details>
