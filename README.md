# Setup EFK stack

![alt text](https://github.com/anjanpaul/EFK-installation/blob/main/output%20image/Screenshot%20from%202022-01-23%2013-57-04.png)


# ElasticSearch

## Step 1: helm initialization
```
helm init

```

## Step 2: Deploying an Elasticsearch Cluster with Helm

```
helm repo add elastic https://helm.elastic.co

```

## Step 2.1: helm update 

```
helm repo update

```
## Step 3 : Next, get the values.yaml file from here. We will modify the value according to our need. Removing all the entries in values.yaml and paste the following entries. Rename the file with esvalues.yaml :

Here we changed the type: ClusterIP to type: LoadBalancer. We are exposing the elasticsearch service externally so that other services can access.

## Step 3.1: Install elasticsearch version 7.13.0 with custom values

```
helm install elasticsearch --version 7.13.0 elastic/elasticsearch -f esvalues.yaml

```
![alt text](https://github.com/anjanpaul/EFK-installation/blob/main/output%20image/elastic%20search1.png)

## Wait for few minutes. After that, you should see all the resources are up and running. Note the elasticsearch service’s external IP. In my case it is 10.116.200.220.

```
kubectl get all -l=chart=elasticsearch 

```
## you can describe your pod also : 

```
kubectl describe pod elasticsearch-master-0

```

## PV and PVC are also deployed for elasticsearch

```
 kubectl get pv,pvc

```
![alt text](https://github.com/anjanpaul/EFK-installation/blob/main/output%20image/pv.png)

## Also note down the image id of the elasticsearch container. In should be 7.13.0

```
kubectl describe pods elasticsearch-master-0 | grep -i image

```


# Fluentd

## Next we will setup fluentd in the cluster. Fluentd will be deployed as daemonset so that it can run in each nodes and collect the pods and nodes logs. Also we need to deploy rbac and service account for fluentd. 

Again I’ve modified the manifest values according to my need. Let’s see the final values. Paste the following snippet in fluentd-ds-rbac.yaml  file

Befor Run fluentd-ds-rbac.yaml change some line in this file

At line 68, I’ve put the elasticsearch service IP got from load balancer service.

At line 89-90, since systemd is not running in the container, we are disabling sytemd conf for fluentd

At line 97-112, I’ve changed the volume mounts so that fluentd collect the nodes and containers logs simultaneously

## Finally, deploying the fluentd

```
kubectl create -f fluentd-ds-rbac.yaml

```

Wait for a few minutes to deploy. Finally you should see all the pods are deployed in the nodes in kube-system namespace.

```
kubectl -n kube-system get all -l=k8s-app=fluentd-logging 

```
![alt text](https://github.com/anjanpaul/EFK-installation/blob/main/output%20image/fluentd.png)

# Kibana

## Lastly, we will deploy Kibana for data visualization. Again we will grab the values from here and modify according to our need. I’ve changed some values in values.yaml file and here is the final modification. Paste the following entries in kivalues.yaml file

At line 2, put the elasticsearch service IP.

At line 7, we explicitly define the image version to be 7.13.0 as our elasticsearch image have the same version. It is important to mention the version. Otherwise, version 8.0.0 will be deployed and EFK stack won’t work properly.

At line 13 & 16, I’ve set the memory 1 GB. If you have plenty of memory left in your nodes, the default value 2 GB is fine.

At line 18, health checker endpoint is changed to /api/status otherwise health checker might fail

Finaly at at line 23, service type changed to LoadBalancer so that we can access Kibana’s dashboard at port 5601

## Install Kibana with Helm along with custom values

```
helm install kibana --version 7.13.0 elastic/kibana -f kivalues.yaml

```

## After a few minutes, all the resources should be deployed and up and running.

```
kubectl get all -l=app=kibana

```

![alt text](https://github.com/anjanpaul/EFK-installation/blob/main/output%20image/kibana.png)

Note down the external ip. In my case it is 10.116.200.221.

Now you should be able to visit Kibana’s dashboard at http://10.116.200.221:5601 😎

## Setup index in Kibana for logs

Go to stack management from left menu

![alt text](https://github.com/anjanpaul/EFK-installation/blob/main/output%20image/efk-03.png)

Select index patterns > Create new index pattern

![alt text](https://github.com/anjanpaul/EFK-installation/blob/main/output%20image/efk-04.png)

Since fluentd followed the logstash format, create the index logstash-*   to capture the logs coming from cluster

![alt text](https://github.com/anjanpaul/EFK-installation/blob/main/output%20image/efk-05.png)

Finally put  @timestamp in time field and create the index pattern

![alt text](https://github.com/anjanpaul/EFK-installation/blob/main/output%20image/efk-06.png)

Now go to Discover from left, you should see the logs

![alt text](https://github.com/anjanpaul/EFK-installation/blob/main/output%20image/efk-07.png)

## Testing the setup

Let’s test the stack by deploying a simple hellopod which just counts.

```
cat <<EOF | kubectl apply -f -                                                    
apiVersion: v1
kind: Pod
metadata:
 name: hellopod
spec:
 containers:
 - name: count
   image: busybox
   args: [/bin/sh, -c,
           'i=0; while true; do echo "$i: Hello from the inside"; i=$((i+1)); sleep 1; done']
EOF

```

See the logs

```
$kubectl logs hellopod -f         

```
Now in Kibana, if you search for kubernetes.pod_name.keyword: hellopod and filter with log and other fields from left, you should see the same logs in Kibana dashboard along with other informations. How cool is that 😃

![alt text](https://github.com/anjanpaul/EFK-installation/blob/main/output%20image/efk-10.png)