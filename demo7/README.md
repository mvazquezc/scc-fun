# **Pod Security Admission**

In these demos we are going to show how Pod Security Admission (PSA) is configured in OCP 4.14 and what options you have as an admin to manage the Pod Security Standards configured for your namespaces.

These demos expect you to already know what PSA is and how it's used, you can refer to the official docs, or read [this blog](https://linuxera.org/working-with-pod-security-standards/).

## **Hands-on Demo 1**

In OCP 4.14 we have the PSA admission configured to apply the following standards to all namespaces (excluding the ones starting by openshift-*):

* Enforce: `privileged`
* Audit: `restricted`
* Warn: `restricted`

You can get the current configuration on your cluster by running the following command:

~~~sh
oc -n openshift-kube-apiserver get $(oc -n openshift-kube-apiserver get cm -o name| grep ^configmap/config | sort -V | tail -1) -o jsonpath='{.data.config\.yaml}' | jq '.admission.pluginConfig.PodSecurity.configuration.defaults'
~~~

When we create a namespace these are the labels we will find related to PSA:

1. Create the namespace

    ~~~sh
    oc create namespace test-psa
    ~~~

2. Get the labels

    ~~~sh
    oc get ns test-psa -o yaml | grep pod-security
    ~~~

    > **NOTE**: Not defining value for enforce equals not enforcing.

    ~~~sh
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.24
    ~~~

When we try to create a workload that won't be allowed by the restricted PSA we will get a warning (caused by the warn mode):

1. Create a workload violating the restricted Pod Security Standard (PSS):

    > **NOTE**: In recent OCP versions the SCC mutation controller will add the required securityContext for Pods created via workload objects like Deployments to match the `restricted` PSA. That's why we create a Pod directly, so you can see the warning.

    ~~~sh
    cat <<EOF | oc -n test-psa apply -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: go-app
    spec:
      containers:
        - image: quay.io/mavazque/reversewords:latest
          name: reversewords
          resources: {}
    EOF
    ~~~

    ~~~sh
    Warning: would violate PodSecurity "restricted:v1.24": allowPrivilegeEscalation != false (container "reversewords" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "reversewords" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "reversewords" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "reversewords" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
    deployment.apps/go-app created
    ~~~

We can see that above workload violates the restricted PSS because:

* It doesn't define `allowPrivilegeEscalation: false` in its security context.
* It doesn't drop all capabilities in its security context.
* It doesn't set the `runAsNonRoot: true` in its security context.
* It doesn't set the default seccomp profile in its security context.

In order to be compliant with the `restricted` PSS we need to change the Pod to something like this:

~~~sh
cat <<EOF | oc -n test-psa apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: go-app-2
spec:
  containers:
    - image: quay.io/mavazque/reversewords:latest
      name: reversewords
      resources: {}
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
EOF
~~~

## **Hands-on Demo 2**

In OCP a new controller has been developed to synchronize the Pod Security Standards configured in a namespace with the SCCs available to workloads in that namespace. In OCP 4.14 this controller modifies the `warn` and `audit` modes. In future OCP releases it will also modify the `enforce` mode.

The way this controller works is by introspecting the service account permissions to use SCCs and maps these SCCs to Pod Security Standards. Based on the SCC configuration the controller will set the value of the different PSA modes that allow the workloads to run in the namespace without triggering warnings or audit logs.

We will see how this controller works and how we can disable it for some of our namespaces.

### **Allow controller to modify the PSA modes**

1. Create a namespace

    ~~~sh
    oc create namespace psa-auto-modes
    ~~~

2. Get the labels

    ~~~sh
    oc get namespace psa-auto-modes -o yaml | grep pod-security
    ~~~

    > **NOTE**: We got the default values configured for OCP 4.14

    ~~~sh
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.24
    ~~~

3. Create a ServiceAccount in the namespace

    ~~~sh
    oc -n psa-auto-modes create serviceaccount test-user
    ~~~

4. Provide access to the privileged SCC

    ~~~sh
    oc -n psa-auto-modes adm policy add-scc-to-user privileged -z test-user
    ~~~

5. Get the labels

    ~~~sh
    oc get namespace psa-auto-modes -o yaml | grep pod-security
    ~~~

    > **NOTE**: Since we have an SA with access to the SCC privileged, the controller automatically changed the values for the PSA modes to `privileged` (the PSS that resembles the privileged SCC the most).

    ~~~sh
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: privileged
    pod-security.kubernetes.io/warn-version: v1.24
    ~~~

6. Cleanup

    ~~~sh
    oc delete ns psa-auto-modes
    ~~~

### **Stop controller from modifying the PSA modes**

1. Create a namespace

    ~~~sh
    oc create namespace psa-manual-modes
    ~~~

2. Get the labels

    ~~~sh
    oc get namespace psa-manual-modes -o yaml | grep pod-security
    ~~~

    > **NOTE**: We got the default values configured for OCP 4.11

    ~~~sh
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.24
    ~~~

3. Add the label to stop the controller from updating the PSA modes based on the SA access to SCCs:

    ~~~sh
    oc label namespace psa-manual-modes security.openshift.io/scc.podSecurityLabelSync=false
    ~~~

4. Create a ServiceAccount in the namespace

    ~~~sh
    oc -n psa-manual-modes create serviceaccount test-user
    ~~~

5. Provide access to the privileged SCC

    ~~~sh
    oc -n psa-manual-modes adm policy add-scc-to-user privileged -z test-user
    ~~~

6. Get the labels

    ~~~sh
    oc get namespace psa-manual-modes -o yaml | grep pod-security
    ~~~

    > **NOTE**: This time, even if we have an SA with access to the SCC privileged, the controller did not update the values for the different nodes.

    ~~~sh
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.24
    ~~~

7. Cleanup

    ~~~sh
    oc delete ns psa-manual-modes
    ~~~
