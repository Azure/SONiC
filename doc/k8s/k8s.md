k8s integration design document
---

## Overview

[Kubernetes](https://kubernetes.io/) (k8s) is an open source container orchestrator
which automates deployment, scaling and management of containerized applications.
By letting k8s manages the containers running in SONiC, we can manage the fleet of
SONiC switches in a centralized way.

k8s has many features to ease the operation of daily containerized applications.
One example is rolling update. By using k8s, the operator can upgrade all container
running in SONiC image with a simple instruction. Stopping the current container and
starting the new updated container can be done automatically in parallel in the fleet of SONiC switches.

## Glossary

Based on [k8s glossary](https://kubernetes.io/docs/reference/glossary/?fundamental=true)

- Cluster
    - A set of machines, called nodes, that run containerized applications managed by Kubernetes
    - In SONiC use-case, each machine is SONiC switch
- Pod
    - The smallest and simplest Kubernetes object. A Pod represents a set of running containers on your cluster
    - In SONiC use-case, swss container can be deployed as one pod ( as well as syncd, database container etc..)
- Service
    - In k8s, 'service' means a specific feature (API object) of k8s and it is not used as a generic term
    - An API object that describes how to access applications, such as a set of Pods , and can describe ports and load-balancers
    - In SONiC use-case, we won't use this feature at least for the first phase since most of the Pods will be deployed with 'hostNetwork: true' which doesn't work with 'service'
        - https://kubernetes.io/docs/concepts/configuration/overview/#services
- Deployment
    - In k8s, 'deployment' means a specific feature (API object) of k8s and it is not used as a generic term
    - An API object that manages a replicated application
    - By using deployment, you can maintain a stable set of replica Pods running at any given time. Also it enables rolling update and changing the number of the replica
    - In SONiC use-case, we won't use this feature at least for the first phase since most of the Pods needs to run on every nodes in the cluster. Also the node can't run the same image more than one (e.g We must not deploy more than 1 syncd container in one SONiC switch). To meet this requirement, we can use DaemonSet.
- DaemonSet
    - Ensures a copy of a Pod is running across a set of nodes in a cluster
    - Used to deploy system daemons such as log collectors and monitoring agents that typically must run on every Node
    - In SONiC use-case, most of the containers can be deployed as DaemonSet. By using this feature, k8s will automatically deploy configured Pods to newly added SONiC switches.
- kubelet
    - An agent that runs on each node in the cluster. It makes sure that containers are running in a pod.

## Requirments

- support following two modes
    1. standalone mode (current model): single node deployment.
    2. cluster mode : multiple nodes deployment. k8s controller in remote
- support switching between standalone mode and cluser mode in runtime
    - at least switching from standalone to cluster mode
- support cluster joining mechanism for newly added switch
    - ideally this should be done automatically when a new switch boots up
- support cluster leaving mechanism
- support rolling update and rollbacking of containers runnning inside SONiC
- support various SONiC versions in the cluster
    - don't blindly deploy the same container image to all nodes

## Assumptions

- Use [k3s](https://k3s.io/) for k8s package
    - lightweight
        - network devices typically have less compute resources compared to server. We should reduce the resource comsumption of k8s as much as possible
        - it is said that k3s binary is less than 40MB and needs 512MB of RAM to run
    - [offline support](https://github.com/rancher/k3s/issues/166) is under development
        - in standaone mode, SONiC should work without the Internet access
        - https://github.com/rancher/k3s/issues/166
        - https://github.com/rancher/k3s/pull/241
    - Other alternatives
        - [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) : the most popular way to install vanilla k8s cluster


## Implementation

### standalone mode

We have 3 options to implement standalone mode.

#### 1. keep using the current implementation based on systemd
- PROS:
    - no need for new development
- CONS:
    - the container manegement methods is totally diffent in cluster mode
        - we need to support two mechanism to manage the containers

```
                 ┌───────┐   ┌──────┐ 
                 │       │   │      │ 
                 │systemd│──▶│docker│ 
┌────────────────┤       ├───┤      ├┐
│                └───────┘   └──────┘│
│            SONiC switch            │
│               (node)               │
└────────────────────────────────────┘
```

#### 2. use kubelet manifest feature
- https://kubernetes.io/docs/tasks/administer-cluster/static-pod/
- PROS:
    - no need to think about changing master node to remote node if moving to cluster mode
    - no need to run k8s controller on switch
- CONS:
    - k3s doesn't support running kubelet only without joining to the cluster

```
  ┌──────────┐   ┌───────┐   ┌──────┐ 
  │  static  │   │       │   │      │ 
  │   pod    │──▷│kubelet│──▶│docker│ 
┌─┤definition├───┤       ├───┤      ├┐
│ └──────────┘   └───────┘   └──────┘│
│            SONiC switch            │
│               (node)               │
└────────────────────────────────────┘
```

#### 3. build single node cluster
- PROS
    - can reuse many things which we use in cluster mode
- CONS:
    - needs to figure out how to switch the controller when moving to cluster mode
        - we need to switch from the controller running inside the switch to the remote controller
        - not sure k3s/k8s has a built-in graceful way to do this

```
 ┌─────────────┐                      
 │  resource   │                      
 │ definition  │                      
 │ (YAML file) │                      
 └─────────────┘                      
        │                             
        ▽                             
   ┌─────────┐                        
   │ kubectl │                        
   └─────────┘                        
        │                             
        ▼                             
  ┌──────────┐   ┌───────┐   ┌──────┐ 
  │          │   │       │   │      │ 
  │controller│──▶│kubelet│──▶│docker│ 
┌─┤          ├───┤       ├───┤      ├┐
│ └──────────┘   └───────┘   └──────┘│
│            SONiC switch            │
│               (node)               │
└────────────────────────────────────┘
```

### cluster mode

```
                                                          ┌───────┐   ┌──────┐ 
                                                          │       │   │      │ 
                                              ┌──────────▶│kubelet│──▶│docker│ 
                                              │          ┌┤       ├───┤      ├┐
                                              │          │└───────┘   └──────┘│
                                              │          │    SONiC switch    │
                                              │          │       (node)       │
                                              │          └────────────────────┘
                  ┌───────────────┐           │                                
                  │               │      ┌────┴───┐       ┌───────┐   ┌──────┐ 
  ┌─────────┐     │  controller   │      │   L3   │       │       │   │      │ 
  │ kubectl │────▶│   (master)    │──────┤network ├──────▶│kubelet│──▶│docker│ 
  └─────────┘     │               │      │        │      ┌┤       ├───┤      ├┐
       △          └───────────────┘      └────┬───┘      │└───────┘   └──────┘│
       │                                      │          │    SONiC switch    │
       │                                      │          │       (node)       │
┌─────────────┐                               │          └────────────────────┘
│  resource   │                               │                                
│ definition  │                               │           ┌───────┐   ┌──────┐ 
│ (YAML file) │                               │           │       │   │      │ 
└─────────────┘                               └──────────▶│kubelet│──▶│docker│ 
                                                         ┌┤       ├───┤      ├┐
                                                         │└───────┘   └──────┘│
                                                         │    SONiC switch    │
                                                         │       (node)       │
                                                         └────────────────────┘
```

#### deploy master node
- the master node needs to be deployed in a remote server.
- k3s master can run inside the container
    - https://github.com/rancher/k3s/blob/master/docker-compose.yml

#### join the cluster
- before joining to the cluster, we need to stop the containers which are currently running on the node
since the master will start deploying the same containers in this node
        - [ ] consider which container we should deploy via k8s
            - database, syncd, swss, teamd, lldp, pmon, bgp, dhcp_relay, snmp, radv, telemetry
- after booting up the master node, token will be generated by the master.
- By using that token and the IP address of the master, each switches can join the cluster
    - `sudo k3s agent --server https://myserver:6443 --token ${NODE_TOKEN}`

#### deploying applications
- [daemonset](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) will deploy the pod to all nodes which matches with the condition in the cluster.
If we add a new switch, k8s can automatically deploy the pod to the switch.
- the condition can be set by using [labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) and [node selector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/).
By assigning a label which corresponds to SONiC version of a node, we can deploy specify which container version to use for specific node. Also we could also assign the role of a node (ToR or Spine) and deploy
different application based on the role.
- daemonset also supports rolling update and rollback
    - https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/
    - https://kubernetes.io/docs/tasks/manage-daemon/rollback-daemon-set/
- the deployment starts by feeding the resource definition (YAML file) to the master node.
