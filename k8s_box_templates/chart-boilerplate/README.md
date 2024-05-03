## K8S_BOX GENERAL TEMPLATE EXPLANATION

To create an helm chart and deploy your project in Kubernetes (or microK8s ecc.) you will need obviously a working K8s cluster or microK8s and Helm installed on your machine.

Copy the contents of the *helm-chart-template* folder. Open *values.yaml* and add the components of your deployment in the way explained below.

### Adding the components of your deployment

The *values.yaml* must contain a list of components like this:

```yaml
components:
- name: python
  image: python:latest
  active: true
  ports:
  - port: 8080
```

As you can see it's a list of containers. Components of the list must have a *name* and an *image* value. To consider a component deployable it must have the *active* parameter set to *true*. 

Obviously you can deploy many components as you want simply by adding another component on the *components* list:

```yaml
components:
- name: python
  image: python:latest
  active: true
  ports:
  - port: 8080
- name: nginx
  image: nginx:latest
  active: true
  ports:
  - port: 9090
```

In this way we have two deployments on the cluster. To start and deploy the chart run the command:

```bash
$ helm install <release-name> /path/to/chart/folder
```

### PORTS

As you can see from the example above, if you want to expose a port in the component you can explicit them in the *ports* list. The correct syntax is the following:

```yaml
ports:
- port: 8080
- port: 9090
```

You can also specify the protocol (if not specified, default is TCP) and the hostPort like this:

```yaml
ports:
- port: 8080
  protocol: TCP
  hostPort: 30000
- port: 9090
  hostPort: 30001
  #For Kubernetes limitations the hostPort range has to be between 30000 and 32767
```

### ENVIRONMENT VARIABLES

Environment variables can be expressed like the following example:

```yaml
environment:
- name: ENV_VAR_NAME
  value: 12345
- name: JAVA_HOME
  value: /path/to/some/folder
```

### COMMANDS

The *commands* list express a series of bash commands that will execute in order of definition in the component. The syntax is the following:

```yaml
commands:
- command: cd $HOME/folder
- command: python program.py
```

### VOLUMES

Like compose, *volumes* explicits folders in the host executor machine that will be mounted in the container. You can specify if a volume is a file or a directory. The syntax is:

```yaml
volumes:
- name: volume-name 
  directory: /path/to/the/folder/to/mount
  mountPath: /path/to/the/mount/destination/folder
- name: volume2-name
  file: /path/to/the/file/to/mount.conf
  mountPath: /path/to/the/mount/destination/file.conf

```

By default the template searches in the file system of the host machine. This case is recommended for microK8s or Minikube users. In a multi-machine Kubernetes cluster is recommended to use NFS. To do this you need to specify the NFS server ip in the *nfsIp* field. 

```yaml
nfsIp: 192.168.1.123
```



#### Root directory

If you need to mount (import) a folder or a single file from the host file system (or NFS), you must specify the root directory path via the *rootDirectory* field. The template will search the specified path for folders or files starting by the *rootDirectory*. For example, if you want to mount the following directory

```yaml
volumes:
- name: volume1
  directory: /build/scripts
  mountPath: /etc/build/scripts
```

With a rootDirectory like:

```yaml
rootDirectory: /etc/foo/
```

The template expects to find *scripts* directory in

```
/etc/foo/build/scripts/
```

So all the files and directories that need to be mounted must be all in the rootDirectory, following the path and the pattern mentioned above.

### JOBS

Simplifying, a *job* in kubernetes is a image that run a list of bash commands once. If the command list will fail it executes again till success. The syntax is the following:

```yaml
jobs:
  image: image-that-will-run-the-command
  commands:
  - command: bash command 1
  - command: bash command 2
```

## SUMMARY

A general example of all the previous rules together is the following values.yaml file:

```yaml
components:
- name: ubuntu
  image: ubuntu:latest
  active: true
  ports: 
  - port: 8080
    hostPort: 30001
  - port: 9090
  environment:
  - name: SOME_NUMBER_VAR
  	value: 89
  - name: ANOTHER_ENV_VAR
  	value: hola
  commands:
  - command: apt get update
  - command: apt install nginx
  volumes:
  - name: volume-name 
    directory: /path/to/the/folder/to/mount
    mountPath: /path/to/the/mount/destination/folder
  - name: volume2-name
    file: /path/to/the/file/to/mount.conf
    mountPath: /path/to/the/mount/destination/file.conf
  jobs:
    image: job-image:latest
    commands:
    - command: command that interact with another component
    - command: another bash command

- name: another-component
  image: another-component:latest
  ....... 
  ........

nfsIp: 192.168.1.123 #optional for clusters with nfs
rootDirectory: /var/foo #mandatory if you use volumes to mount directories and files
```

This is a generic example, a complete and working one can be find in the *example* directory that will run multiple components (and relative container) that communicate with each other.