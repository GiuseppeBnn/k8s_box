### F1 TAP Project in Helm template example

To start the example you have to be in the example chart folder (where is located the Chart.yaml file). Then you must have installed obviously an istance of Minikube or a Kubernetes cluster with at least 10 GB of RAM (you can modify RAM limits parameters in value.yaml file in some components). You must have also Helm installed. At this point you can run the following command:

```sh
$ helm install <nome-release> ./
```

Once you have downloaded all the images and started them in the various nodes (or in the single node in the case of Minikube), you can observe the result on Kibana (localhost:30001). Obviously, once you open Kibana you need to load the dashboard configuration file (DASH8.ndjson) in the Saved Objects. Enjoy!