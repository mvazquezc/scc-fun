# **Fun with SCCs**

This repository has some demo labs to help you understand how different SCC subsystems work.

It's highly recommended reviewing the following blogs in order to understand the concepts used in the labs:

* [Container Security - Linux Capabilities and Secure Compute Profiles](https://linuxera.org/container-security-capabilities-seccomp/)
* [Capabilities and Seccomp Profiles on Kubernetes](https://linuxera.org/capabilities-seccomp-kubernetes/)

## **Versions used**

Labs were last tested with OCP v4.11.0.

## **Demo 1**

SCC for workloads, learn how SCCs are accessed, ordered and prioritized for your workloads.

[Start Here](./demo1/README.md)

## **Demo 2**

Seccomp profiles, learn how to create your own seccomp profiles and use them on OpenShift.

[Start Here](./demo2/README.md)

## **Demo 3**

Capabilities, learn what they are and how you can allow/restrict their use on OpenShift.

[Start Here](./demo3/README.md)

## **Demo 4**

SCCs strategies, learn how to work with the different SCC strategies on OpenShift.

[Start Here](./demo4/README.md)

## **Demo 5**

Debugging SCCs Issues, apply your knowledge around SCCs to solve some issues related to SCCs.

[Start Here](./demo5/README.md)

## **Demo 6**

Privilege Escalation bit, learn how to control if your containers can run privilege escalation operators through the use of _no_new_privs_ bit.

[Start Here](./demo6/README.md)
