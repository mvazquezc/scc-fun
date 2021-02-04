# **SCCs Priorities**

In this demo we are going to show a basic scenario of SCCs prioritization that will help you understand how SCCs are ordered and prioritized.

## **Hands-on Demo**

1. Create a namespace for running the demo

    ~~~sh
    NAMESPACE="scc-fun-1"
    oc create ns ${NAMESPACE}
    ~~~
2. Create a service account that will be used as our unprivileged user (we will impersonate it)

    ~~~sh
    oc -n ${NAMESPACE} create sa testuser
    ~~~
3. Give edit role to the `testuser` SA in the namespace

    ~~~sh
    oc -n ${NAMESPACE} adm policy add-role-to-user edit system:serviceaccount:${NAMESPACE}:testuser
    ~~~
4. Give the SA access to the SCC `restricted`
    > **NOTE**: All authenticated users have access to the `restricted` SCC, since it's the default one. We're only giving the SA this access so the SCC gets reported back when using `scc-review` commands. Scc-review command doesn't take into account access to SCCs inherited via a user group, such as `system:authenticated`.
    ~~~sh
    oc -n ${NAMESPACE} adm policy add-scc-to-user restricted system:serviceaccount:${NAMESPACE}:testuser
    ~~~
5. Create a pod definition and store it on /tmp
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
6. Run an scc-review to see which SCC will be assigned to the pod

    ~~~sh
    oc -n ${NAMESPACE} policy scc-review -z system:serviceaccount:${NAMESPACE}:testuser -f /tmp/pod-scc-1.yaml 
    ~~~
    1. As you can see it will be granted the `restricted` SCC
        
        ~~~
        RESOURCE        SERVICE ACCOUNT   ALLOWED BY   
        Pod/pod-scc-1   testuser          restricted     
        ~~~
7. Create the pod (as the non-admin user) and check the SCC assigned to it

    ~~~sh
    oc -n ${NAMESPACE} create -f /tmp/pod-scc-1.yaml --as=system:serviceaccount:${NAMESPACE}:testuser

    oc -n ${NAMESPACE} get pod pod-scc-1 -o yaml | grep "openshift.io/scc"
    ~~~

    1. As you can see, the `restricted` SCC was granted
    
        ~~~
        openshift.io/scc: restricted
        ~~~
8. Let's give access to the `anyuid` SCC to the unprivileged user

    ~~~sh
    oc -n ${NAMESPACE} adm policy add-scc-to-user anyuid system:serviceaccount:${NAMESPACE}:testuser
    ~~~
9. What happens if we run the scc-review again?

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
10. Let's create the pod again to see what SCC does it get this time

    ~~~sh
    oc -n ${NAMESPACE} delete pod pod-scc-1
    oc -n ${NAMESPACE} create -f /tmp/pod-scc-1.yaml --as=system:serviceaccount:${NAMESPACE}:testuser

    oc -n ${NAMESPACE} get pod pod-scc-1 -o yaml | grep "openshift.io/scc"
    ~~~

    1. As you can see, the `anyuid` SCC was granted
    
        ~~~
        openshift.io/scc: anyuid
        ~~~