## K8S_BOX GENERAL TEMPLATE EXPLANATION

To create an helm chart and deploy your project in Kubernetes (or Minikube ecc.) you will need obviously a working K8s cluster or Minikube and Helm installed on your machine.

First of all generate a empty helm chart with the command:

```bash
$ helm create <chart-name>
```

Now remove *values.yaml* file, *templates* and *chart* folder:

```bash
$ rm ./values.yaml
$ rm -rf ./templates ./charts
```

Create from zero a new *templates* folder and a new *values.yaml* file in the chart main directory. Next create a template file *template.yaml* in *templates* directory:

```bash
$ mkdir templates
$ touch values.yaml
$ touch /templates/template.yaml
```

Copy the template file from this repository as it is and save. 

To create your deployments you have to specify components, images, ports, variable ecc.... in the *values.yaml* as descripted below.

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

### VOLUMES and VOLUMEMOUNTS

Like compose, *volumes* explicits folders in the host executor machine that will be mounted (via VolumesMounts) in the container. You can specify if a volume is a file or a directory. The syntax is:

```yaml
volumes:
- name: volume-name
  hostPath: 
    path: /path/to/the/folder/to/mount
    type: Directory
- name: volume2-name
  hostPath:
  	path: /path/to/the/file/to/mount.conf
  	type: File
```

volumeMounts indicates the mount destination path of a predefined Volume like this:

```yaml
volumeMounts:
- name: volume-name  #the name of the volume you want to mount
  mountPath: /path/to/the/mount/destination/folder
- name: volume2-name
  mountPath: /path/to/the/mount/destination/file.conf
```

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
  - name: volume1
    hostPath:
      path: /path/to/a/folder/in/host/machine
      type: Directory
  - name: volume2
    hostPath:
      path: /path/to/a/file/in/host/machine.config
      type: File
  volumeMounts:
  - name: volume1
    mountPath: /mount/path/of/folder
  - name: volume2
  	mountPath: /mount/path/of/file.config
  	
  jobs:
    image: job-image:latest
    commands:
    - command: command that interact with another component
    - command: another bash command

- name: another-component
  image: another-component:latest
  ....... 
  ........
```

This is a generic example, a complete and working one can be find in the *example* directory that will run multiple components (and relative container) that communicate with each other.