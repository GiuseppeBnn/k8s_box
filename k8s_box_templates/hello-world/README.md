## Hello World step by step

You'll find below a step by step guide for a deployment of 3 components that communicate with each other, This example is done with microK8s.

- "python sender" deployment that generates "hello world" strings and sends them to logstash
- "logstash", it receives messages and forwards them to another python receiver through a pipeline
- "python receiver", it receives from logstash messages and print them in console

First of all, you have to install microK8s in your machine. Visit https://microk8s.io/ and install it on Linux, Windows or MacOS.

*MicroK8s contains also Helm for deploying the components, so if you want to use Minikube (or others) you have to install Helm separately (https://helm.sh/docs/intro/install/).*

At this point we have to copy the content of */chart-boilerplate* to a new empty folder, let's call it */hello-chart*. So the folder and its expected content must be like this

```
/hello-chart
  /templates
    template.yml    # principal boilerplate
  Chart.yaml
  values.yaml       # file to modify
```



Done that, from now on the only file that will be modified is *values.yaml*. Let's empty this file and start from scratch.

We start by defining our components list and adding the first component of our deployment. In particular, for every component in the list must be specified the name, the image and the active option to start properly. So let's define a name and a image, in our case we name it 'producer' and use 'python:latest' as image.

```yaml
components:
- name: producer
  image: python:latest
  active: true  #set false if you want helm to ignore this element
```

Now we have the bare minimum to start the deployment. This component (container) has to start a *.py* script file that will be the producer. I'll use this simple python script to generate random helloworld messages and send via tcp to logstash.

```python
# producer.py
import socket
import time
logstash_host = 'logstash'
logstash_port = 9191
def testLogstash():
    while True:
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.connect((logstash_host, logstash_port))
            sock.close()
            break
        except:
            print("Logstash not ready")
            time.sleep(5)
            continue
def sendToLogstash(data):
    print(data, flush=True)
    #print("inviato")
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect((logstash_host, logstash_port))
    sock.sendall(data.encode('utf-8'))
    sock.close()
def main():
    counter = 0
    testLogstash() #readiness check
    while True:
        # Generate "hello-world" message
        message = 'Hello, world!'+str(counter)
        counter += 1
        sendToLogstash(message)
        time.sleep(1)
if __name__ == "__main__":
    print("Starting producer...")
    main()
```

Save this file in a folder, i'll call it */hello-world-root-dir/producer/producer.py* (absolute path). Later it'll be explained how to and where place this file to allow Kubernetes (microK8s) to find and to mount it.

Now the producer component needs to have this script in its container to start it. To do that we'll use *volumes* to take the correct folder and mount it into the desired location where it'll be started. Kubernetes will look for all the *directories* and *files* specified in *volumes* starting from the <u>rootDirectory</u>. This is a directory in your machine where all the files and folders to be mounted need to be, in my case i'll set this folder */hello-world-root-dir*. Now specify in *volumes* the name of the first volume of the list, the *directory* path and the mount destination path in the container. Since i placed the *producer* folder in */hello-world-root-dir/producer/*, i'll write in the *directory* field only */producer* because, as mentioned before, the *rootDirectory* (*/hello-world-root-dir*) acts as an absolute path for Kubernetes (microK8s). Next, in the *mountPath* field we specify the destination mount path in the container file system.

```yaml
rootDirectory: /hello-world-root-dir
components:
- name: producer
  image: python:latest
  volumes:
  - name: producer_folder
    directory: /producer/
    mountPath: /app/
```

So in the container /app/ folder we'll find *producer.py*. Now this scripts need to be started by a bash command. To specify a command, or a list of commands, we need to use the *commands* list and the *command* item of the list like this:

```yaml
rootDirectory: /hello-world-root-dir
components:
- name: producer
  image: python:latest
  volumes:
  - name: producer_folder
    directory: /producer/
    mountPath: /app/
  commands:
  - command: cd /app
  - command: python producer.py
```

Now this component seems to be ready. Let's specify the second one. Logstash will create a pipeline that listens to port 9191 (see producer script) with TCP protocol and forwards messages to the consumer listening to port 9292 TCP.

Like before we must specify the image and the name. The latter, in every component, is also the hostname of the component and so, like docker, it will be find by the Kubernetes dns by the component's name. In the producer script we specified the logstash_host value to logstash, so we need to call this component *logstash*:

```yaml
rootDirectory: /hello-world-root-dir
components:
- name: producer
  image: python:latest
  volumes:
  - name: producer_folder
    directory: /producer/
    mountPath: /app/
  commands:
  - command: cd /app
  - command: python producer.py
###
- name: logstash
  image: logstash:8.7.1
  ports:
  - port: 9191
    protocol: TCP
  
```

Logstash pipeline needs to listen to a specified port, so we need to open 9191 port using the *ports* list field like above. 

Now we'll create the logstash.conf file that will be mounted later as a volume. *logstash.conf*:

```conf
input{
    tcp{
        port => 9191    
    }
}
output{
    tcp{
        host => "consumer"
        port => 9292
    }
}
```

And the *pipelines.yml* file that tells logstash where to find the pipeline:

```
- pipeline.id: hello-world-pipeline
  path.config: "/usr/share/logstash/pipeline/logstash.conf"
```

We can use *volumes* to mount also specific files by using *file* field instead of *directories*. Like this:

```yaml
rootDirectory: /hello-world-root-dir #absolute path to the dir
components:
- name: producer
  image: python:latest
  active: true
  volumes:
  - name: producer_folder
    directory: /producer/
    mountPath: /app/
  commands:
  - command: cd /app
  - command: python producer.py
###
- name: logstash
  image: logstash:8.7.1
  active: true
  ports:
  - port: 9191
    protocol: TCP
  volumes:
  - name: logstash-conf   #don't use dots ('.') in the name, it will cause errors
    file: /logstash/logstash.conf
    mountPath: /usr/share/logstash/pipeline/logstash.conf
  - name: pipelines
  	file: /logstash/pipelines.yml
  	mountPath: /usr/share/logstash/config/pipelines.yml
  environment:
  - name: LS_JAVA_OPTS
    value: -Xms1g -Xmx1g
```

*environment* field list can be used to specify environment variables, the syntax is really simple like above. We pass this variable to maximize the memory used by the component. 

Let's now write the last component, the consumer. Like the producer element, this component needs to load a file volume to start (python script) and furthermore it needs a port open where it will receive the messages from logstash. The name needs to be like the hostname of the receiver specified in logstash.conf to be found. First of all create a simple *consumer.py*:

```python
#consumer.py
import socket
def main():
    host = 'consumer' # hostname 
    port = 9292 
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.bind((host, port))
        print(f"In ascolto su {host}:{port}", flush=True)
        s.listen()
        conn, addr = s.accept()
        with conn:
            print(f"Connesso a {addr}")
            while True:
                data = conn.recv(1024)
                if not data:
                    break
                print(f"Messaggio ricevuto da Logstash: {data.decode('utf-8')}", flush=True)

if __name__ == "__main__":
    main()
```

Save this in the correct path like the producer, i saved this in */hello-world-root-dir/consumer/consumer.py*.

At this point values.yaml file will be like this:

```yaml
rootDirectory: /hello-world-root-dir #absolute path to the dir
components:
- name: producer
  image: python:latest
  active: true
  volumes:
  - name: producer_folder
    directory: /producer/
    mountPath: /app/
  commands:
  - command: cd /app
  - command: python producer.py
###
- name: logstash
  image: logstash:8.7.1
  active: true
  ports:
  - port: 9191
    protocol: TCP
  volumes:
  - name: logstash-conf   #don't use dots ('.') in the name, it will cause errors
    file: /logstash/logstash.conf
    mountPath: /usr/share/logstash/pipeline/logstash.conf
  - name: pipelines
  	file: /logstash/pipelines.yml
  	mountPath: /usr/share/logstash/config/pipelines.yml
  environment:
  - name: LS_JAVA_OPTS
    value: -Xms1g -Xmx1g
 ###
 - name: consumer
   image: python:latest
   active: true
   ports:
   - port: 9292
     protocol: TCP
   volumes:
   - name: consumer-file
     file: /consumer/consumer.py
     mountPath: /app/consumer.py
   commands:
   - command: cd /app
   - command: python consumer.py
```

Now we are ready to deploy, but first we have to check if microK8s is ready. Run:

```bash
$ microk8s status --wait-ready
```

to verify if all is ready. Then, deploy using helm:

```bash
$ microk8s helm install hello-world ./
```

Kubernetes (microk8s) now is pulling images and starting all the containers (components). If it returns an error it means that something in the logic of the deployment or in the syntax of the values.yaml file isn't correct, please read again the instructions. 

To verify the deployments status and take the deployments names run:

```bash
$ microk8s kubectl get pods
```

instead, if you want to see more details, run the following command with the deployment name take before:

```bash
$ microk8s kubectl describe pod <deployment-name-seen-above>
```

Finally, to verify and see the messages in the receiver use:

```bash
$ microk8s kubectl attach <deployment-name-seen-above>
```



This is a simple Hello World deployment written to help the understanding of how K8s_box works. There more fields that were not used in this example which you could find in the README.md in the boilerplate folder. Don't forget to set the root Directory in the correct position!

ENJOY