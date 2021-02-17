# **SCCs Access and Priorities**

In this demo we are going to show a basic scenarios of SCCs access and prioritization that will help you understand how SCCs are accessed, ordered and prioritized.


## **Hands-on Demo 1**

In this demo we will see how a ServiceAccount (SA) can get access to use an SCC by directly granting access to the SCC to the SA.

1. Create a namespace for running the demo

    ~~~sh
    NAMESPACE="scc-fun"
    oc create ns ${NAMESPACE}
    ~~~
2. Create a ServiceAccount

    ~~~sh
    oc -n ${NAMESPACE} create serviceaccount testsa1
    ~~~
3. Grant access to the `anyuid` SCC directly to the SA

    ~~~sh
    oc -n ${NAMESPACE} adm policy add-scc-to-user anyuid system:serviceaccount:${NAMESPACE}:testsa1
    ~~~
4. Check who can use the `anyuid` SCC
    ~~~sh
    oc adm policy who-can use scc anyuid
    ~~~

    > **NOTE**: As you can see the SA testsa1 has access to use the scc.
    ~~~
    Namespace: scc-fun
    Verb:      use
    Resource:  securitycontextconstraints.security.openshift.io

    Users:  system:admin
            <OUTPUT_OMITTED>
            system:serviceaccount:scc-fun:testsa1
    Groups: system:cluster-admins
            system:masters
    ~~~

## **Hands-on Demo 2**

In this demo we will see how a ServiceAccount can get access to use an SCC (anyuid). This can be done by creating a Role that allow to use the security.openshift.io API, the resource securityContextConstraint and the specific name of the resource: anyuid. Then, bind the Role to the SA by means of a Rolebinding.

1. Create a new ServiceAccount

    ~~~sh
    oc -n ${NAMESPACE} create serviceaccount testsa2
    ~~~
2. Create a Role that grants access to use the anyuid scc

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: use-anyuid
    rules:
    - apiGroups:
      - security.openshift.io
      resources:
      - securitycontextconstraints
      resourceNames:
      - anyuid
      verbs:
      - use
    EOF
    ~~~
3. Create a RoleBinding that grants `use-anyuid` role to our SA `testsa2`

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: use-anyuid-rb
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: use-anyuid
    subjects:
    - kind: ServiceAccount
      name: testsa2
      namespace: ${NAMESPACE}
    EOF
    ~~~
4. Check who can use the `anyuid` SCC
    ~~~sh
    oc adm policy who-can use scc anyuid
    ~~~

    > **NOTE**: As you can see the SA testsa2 has access to use the scc.
    ~~~
    Namespace: scc-fun
    Verb:      use
    Resource:  securitycontextconstraints.security.openshift.io

    Users:  system:admin
            <OUTPUT_OMITTED>
            system:serviceaccount:scc-fun:testsa1
            system:serviceaccount:scc-fun:testsa2
    Groups: system:cluster-admins
            system:masters
    ~~~
5. Assigning SCCs to SAs using Roles is the prefered way to go. During the upcoming demos we will gran direct access to make things easier though.

## **Hands-on Demo 3**

1. Create a service account that will be used as our unprivileged user (we will impersonate it)

    ~~~sh
    oc -n ${NAMESPACE} create sa testuser
    ~~~
2. Give edit role to the `testuser` SA in the namespace

    ~~~sh
    oc -n ${NAMESPACE} adm policy add-role-to-user edit system:serviceaccount:${NAMESPACE}:testuser
    ~~~
3. Give the SA access to the SCC `restricted`
    > **NOTE**: All authenticated users have access to the `restricted` SCC, since it's the default one. We're only giving the SA this access so the SCC gets reported back when using `scc-review` commands. Scc-review command doesn't take into account access to SCCs inherited via a user group, such as `system:authenticated`.
    ~~~sh
    oc -n ${NAMESPACE} adm policy add-scc-to-user restricted system:serviceaccount:${NAMESPACE}:testuser
    ~~~
4. Create a pod definition and store it on /tmp
    ~~~sh
    cat <<EOF > /tmp/pod-scc-1.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-scc-1
    spec:
      serviceAccountName: testuser
      containers:
      - image: quay.io/mavazque/reversewords:latest
        name: test-pod
        resources: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
    status: {}
    EOF
    ~~~
5. Run an scc-review to see which SCC will be assigned to the pod

    ~~~sh
    oc -n ${NAMESPACE} policy scc-review -z system:serviceaccount:${NAMESPACE}:testuser -f /tmp/pod-scc-1.yaml 
    ~~~
    1. As you can see it will be granted the `restricted` SCC
        
        ~~~
        RESOURCE        SERVICE ACCOUNT   ALLOWED BY   
        Pod/pod-scc-1   testuser          restricted     
        ~~~
6. Create the pod (as the non-admin user) and check the SCC assigned to it

    ~~~sh
    oc -n ${NAMESPACE} create -f /tmp/pod-scc-1.yaml --as=system:serviceaccount:${NAMESPACE}:testuser

    oc -n ${NAMESPACE} get pod pod-scc-1 -o yaml | grep "openshift.io/scc"
    ~~~

    1. As you can see, the `restricted` SCC was granted
    
        ~~~
        openshift.io/scc: restricted
        ~~~
7. Let's give access to the `anyuid` SCC to the unprivileged user

    ~~~sh
    oc -n ${NAMESPACE} adm policy add-scc-to-user anyuid system:serviceaccount:${NAMESPACE}:testuser
    ~~~
8. What happens if we run the scc-review again?

    ~~~sh
    oc -n ${NAMESPACE} policy scc-review -z system:serviceaccount:${NAMESPACE}:testuser -f /tmp/pod-scc-1.yaml 
    ~~~
    1. The `anyuid` was placed before the `restricted` SCC
        
        ~~~
        RESOURCE        SERVICE ACCOUNT   ALLOWED BY   
        Pod/pod-scc-1   testuser          anyuid       
        Pod/pod-scc-1   testuser          restricted   
        ~~~
    2. Our pod spec doesn't have anything that will require it to run with `anyuid`, why `anyuid` has preference over restricted?

        ~~~sh
        oc get scc restricted anyuid
        NAME         PRIV    CAPS         SELINUX     RUNASUSER        FSGROUP     SUPGROUP   PRIORITY     READONLYROOTFS   VOLUMES
        restricted   false   <no value>   MustRunAs   MustRunAsRange   MustRunAs   RunAsAny   <no value>   false            ["configMap","downwardAPI","emptyDir","persistentVolumeClaim","projected","secret"]
        anyuid       false   <no value>   MustRunAs   RunAsAny         RunAsAny    RunAsAny   10           false            ["configMap","downwardAPI","emptyDir","persistentVolumeClaim","projected","secret"]
        ~~~
    3. The SCC `anyuid` has a priority of 10 while restricted has no priority, that means `anyuid` will be prefered even if the pods doesn't need it
    
9. Let's create the pod again to see what SCC does it get this time

    ~~~sh
    oc -n ${NAMESPACE} delete pod pod-scc-1
    oc -n ${NAMESPACE} create -f /tmp/pod-scc-1.yaml --as=system:serviceaccount:${NAMESPACE}:testuser
    ~~~

    ~~~sh
    oc -n ${NAMESPACE} get pod pod-scc-1 -o yaml | grep "openshift.io/scc"
    ~~~
    
    > **NOTE**: the `anyuid` SCC was granted

    ~~~
    openshift.io/scc: anyuid
    ~~~

     > **NOTE**: An easier way to check the SCC assigned to your pod, specially if you have multiple pods with likely different SCC assigned to it, it is formating the output using custom-columns. For instance, you can just show the name of the pod and the SCC applied without require access to the yaml definition or even the name of the pod:
    
    ~~~sh
    oc -n ${NAMESPACE} get pod -o 'custom-columns=NAME:metadata.name,APPLIED SCC:metadata.annotations.openshift\.io/scc'
    ~~~

    ~~~
    NAME        APPLIED SCC
    pod-scc-1   anyuid
    ~~~