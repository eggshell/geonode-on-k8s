## Overview

Currently, there is no official path for GeoNode to be deployed onto a kubernetes cluster, whether they be managed by a cloud service (IBM Kubernetes Service, Google Kubernetes Engine, Azure Kubernetes Service, etc) or rolled in-house. Kubernetes has more or less emerged as the de facto container orchestration system, so providing this deployment method would be a good step towards providing more options for real production GeoNode.

I have gone through the process of converting the `docker-compose.yml` into some corresponding kubernetes manifest yamls (#3911) which should be platform-agnostic and can serve as a base for maturation into a fully baked production deployment solution. This issue can serve as a place to discuss kubernetes concepts, the architecture for the GeoNode kubernetes deployment as it stands, and keep track of to-do's.

## Architecture

This is the architecture of the GeoNode kubernetes deployment as it stands right now.

![geonode_k8s_diagram](https://github.com/eggshell/geonode-on-k8s/blob/master/img/geonode_k8s_diagram.png)

## Concepts

### Pods

A pod in kubernetes is a group of one or more containers. They share storage and network, are co-located and co-scheduled, and run in a shared context (credit: k8s official docs on podS). The list of pods in the GeoNode deployment is:

* Geonode (nginx)
* Geoserver
* rabbitmq
* django + dind
* celery + dind
* postgresql

Each pod in this deployment contains 1 container, with the exception of the django and celery pods, which each have a `docker-in-docker` pod, which should be able to go away with more modification. More on that below.

### Services and Service Discovery

Kubernetes services target pods and expose their functionality to other pods, providing a policy which pods use to communicate with one another. This is usually done by exposing a port on a container, the same way you would open a port on traditional infrastructure.

Services are assigned a name, and this combination of service name and port are injected into DNS records cluster-wide so these names can be resolved.

Example service definition:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: geonode
spec:
  selector:
    app: geonode
  ports:
  - name: geonode
    protocol: TCP
    port: 80
```

### Deployments

Each pod is defined in a kubernetes deployment. A deployment is a resource type in kubernetes which specifies desired state for a pod. The deployment makes sure a pod is always running. Has a pod crashed? Restart it. Want to scale up/down the number of replicated containers in a pod? Go ahead and do it. Has the desired state changed? Terminate the old pod and spin up a new one.

Example deployment defintion:

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: geonode
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: geonode
    spec:
      containers:
      - image: geonode/nginx:geoserver
        name: nginx4geonode
        ports:
        - containerPort: 80
        stdin: true
      restartPolicy: Always
```

### Persistent Volumes

You _could_ provision a container volume in the spec above, but in the case of GeoNode, we want this data to have a lifecycle independent of any individual pod that uses it. Say a pod's spec changes. It would need to be terminated and recreated, and if its volumes are provisioned with the pod, then that data is lost. The PersistentVolume resource type allows us to provision some storage space in a cluster for this use case.

Example PersistentVolume definition:

```yaml
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: geonode-geoserver-data-dir
  labels:
    type: local
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/mnt/geoserver-data"
```

PersistentVolumeClaims are bound to PersistentVolumes and referenced by pods.

Example PersistentVolumeClaim definition:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: geonode-geoserver-data-dir
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
```

Then, a deployment spec's volumes section would reference this PersistentVolumeClaim:

```yaml
...
volumes:
- name: geonode-geoserver-data-dir
  persistentVolumeClaim:
    claimName: geonode-geoserver-data-dir
```

Finally, a deployment spec's container section defines where this volume should be mounted.

```yaml
...
image: eggshell/geonode:latest
imagePullPolicy: Always
name: django4geonode
stdin: true
volumeMounts:
- name: geonode-geoserver-data-dir
  mountPath: /geoserver_data/data
```

### Ingress

Kubernetes Ingress resources manage external access to a service or services deployed to a kubernetes cluster (typically HTTP). Ingress can provide load balancing, SSL termination, and name-based virtual hosting. Right now I just have it set to provide external access, so it is pretty simple.

```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: geonode-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    ingress.bluemix.net/client-max-body-size: "100m"
spec:
  tls:
  - hosts:
    - eggshell.us-south.containers.appdomain.cloud
    secretName: eggshell
  rules:
  - host: eggshell.us-south.containers.appdomain.cloud
    http:
      paths:
      - path: /
        backend:
          serviceName: geonode
          servicePort: 80
```

The `annotations` section here allows for the ingress resource to be configured at creation time to correctly set the `client-max-body-size` nginx parameter so layer uploads in GeoNode are possible.

## Hacks Used

### Init Containers

Init Containers run before application containers are started, can have their own container image, and typically contain setup scripts or utilities not present in the application container image. I used an Init Container in the geoserver deployment definition to ensure the `geoserver_data` persistent volume has its initial data. This Init Container runs anytime a `geoserver` deployment is configured, but the of the `-n` flag in cp ensures data is not overwritten if a given file is present. This ensures fault tolerance on the part of th egeoserver deploymen persistent volume has its initial data. This Init Container runs anytime a `geoserver` deployment is configured, but the of the `-n` flag in cp ensures data is not overwritten if a given file is present. This ensures fault tolerance on the part of the geoserver deployment.

```yaml
initContainers:
  - name: data-dir-conf
    image: geonode/geoserver_data:latest
    command: ["cp", "-nr", "/tmp/geonode/downloaded/data", "/geoserver_data"]
    volumeMounts:
    - mountPath: /geoserver_data/data
      name: geonode-geoserver-data-dir
```

### docker-in-docker

The Django and Celery deployments each have a `docker-in-docker` or `dind` container. The `docker.sock` of the `dind` container is then bound to the other application container present in the pod so that it may run the `codenvy/che-ip` container and not crash.

I really just included this as a stopgap to get GeoNode running it kubernetes. It seems to be endemic to the `docker-compose` local deployment, but I wanted more information and discussion about this topic before removing all the plumbing. Thoughts?

```yaml
- name: dind
  image: docker:dind
  securityContext:
    privileged: true
  args:
    - dockerd
    - --storage-driver=overlay2
    - -H unix:///var/run/docker.sock
  volumeMounts:
  - name: varlibdocker
    mountPath: /var/lib/docker
  - name: rundind
    mountPath: /var/run/
```

```yaml
volumes:
- name: varlibdocker
  emptyDir: {}
- name: rundind
  hostPath:
    path: /var/run/dind/
```

## TODO

- [ ] Get running on minikube (should just be ingress config)
- [ ] Think about if we want to template using Helm for different deployment
      scenarios (local on minikube, managed k8s cluster, different providers,
      etc)
- [ ] Write some how-to docs for getting k8s deployments going
- [ ] See if we can remove the `dind` stuff, as it doesn't appear to communicate
      with other services or do any realy work in kubernetes.
- [ ] Maybe consolidate docker images? It's a little confusing how all of that
      is piped through.

What am I missing? Thanks for reading!
