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
          - image: quay.io/mavazque/reversewords:latest
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

1. Create a new namespace for running our tests:

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

We have a workload that requires running with a given UID (1024). We create the deployment but pods are not being created. 

1. Create a new namespace for running our tests:

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