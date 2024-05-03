### F1 TAP Project in Helm template example

To start the example you have to be in the example chart folder (where is located the Chart.yaml file). Then you must have installed obviously an istance of Minikube or a Kubernetes cluster with at least 10 GB of RAM (you can modify RAM limits parameters in value.yaml file in some components). You must have also Helm installed. 

IMPORTANT: copy (or move) the SharedFS folder and its content in /SharedFS. In values.yaml the rootDirectory is /SharedFS and Kubernetes will look for the directories and files to mount in /SharedFS (absolute path). You obviously can change rootDirectory path if you want to the new location of the folder.

At this point you can run the following command:

```sh
$ microk8s helm install <nome-release> ./
```

For example, if i want to call the release f1tap, i'll type:

```bash
$ microk8s helm install f1tap ./
```

Once microk8s (or Kubernetes ecc..) has downloaded all the images and started them in the various nodes (or in the single node in the case of microk8s), you can observe the result on Kibana (localhost:30001). Obviously, once you open Kibana you need to load the dashboard configuration file (DASH8.ndjson) in the Saved Objects.

 Enjoy!