# How to install Service Mesh with Istio in k8s

**This step by step show how to install service mesh with istio in an kubernetes cluster already created.
**

##Requirements:

**Have a k8s Cluster already created in the cloud** → [Here an Example](https://github.com/diegokalpa/website-GCP-TERRA "Here an Example")](https://github.com/diegokalpa/website-GCP-TERRA)

**Helm charts, preferably latest version.**

In this example we will use GCP as a cloud with the GKE service,  is necessary to have a gcloud connection to the cluster → [<[install gcloud here](https://cloud.google.com/sdk/docs/install "install gcloud here")>](https://cloud.google.com/sdk/docs/install)

Comands used in MAC.
Shortcut used k = kubectl

## Install Helm Latest Version (In MAC)

`$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh`

## Download Istio

`curl -L https://istio.io/downloadIstio | sh -
cd istio-<version-descargada>`

Export Istioctl path:

`export PATH=$PWD/bin:$PATH`

## Installing Istio
`
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y`

you will receive some like this:

`✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Installation complete                                                                                                                        Making this installation the default for injection and validation.

Thank you for installing Istio 1.14.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/yEtCbt45FZ3VoDT5A```

And checking the namespace:

```bash
k get ns
NAME              STATUS   AGE
default           Active   20m
istio-system      Active   2m46s
kube-node-lease   Active   20m
kube-public       Active   20m
kube-system       Active   20m
```

**Add label to default namespace, this is the order to inject the sidecar to any pod:
**

`kubectl label namespace default istio-injection=enabled
`
## Adding a test application to test Istio

Inside Istio path:

`kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
`
You will receive some like this:

```bash
k get pods
NAME                              READY   STATUS              RESTARTS   AGE
details-v1-7d88846999-vq58f       1/1     Running             0          56s
productpage-v1-7795568889-9llck   1/1     Running             0          51s
ratings-v1-754f9c4975-hhwm8       1/1     Running             0          54s
reviews-v1-55b668fc65-fzk7m       1/1     Running             0          53s
reviews-v2-858f99c99-dd44h        1/1     Running             0          53s
reviews-v3-7886dd86b9-8pjkl       0/1     ContainerCreating   0          52s

```
(if the status READY in any Pod is 2/2 skip this step)
IMPORTANT: If pods in column READY are in 1/1 it means that Istio was not installed in pods, if this happens execute a rollout restart to each pod like this:, 

```bash
k get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
details-v1       1/1     1            1           4m44s
productpage-v1   1/1     1            1           4m39s
ratings-v1       1/1     1            1           4m42s
reviews-v1       1/1     1            1           4m41s
reviews-v2       1/1     1            1           4m41s
reviews-v3       1/1     1            1           4m40s
```

➜  istio-1.14.3 k rollout restart deployment details-v1 productpage-v1 ratings-v1 reviews-v1 reviews-v2 reviews-v3

```bash
deployment.apps/details-v1 restarted
deployment.apps/productpage-v1 restarted
deployment.apps/ratings-v1 restarted
deployment.apps/reviews-v1 restarted
deployment.apps/reviews-v2 restarted
deployment.apps/reviews-v3 restarted
```

If Istio was correctly installed the Ready column will show 2/2:

```bash
k get pods
NAME                             READY   STATUS    RESTARTS   AGE
details-v1-5b76489879-l8bcn      2/2     Running   0          55s
productpage-v1-9b9fcf4cc-2978k   2/2     Running   0          55s
ratings-v1-6699d86687-652xb      2/2     Running   0          55s
reviews-v1-86697458ff-fxqdd      2/2     Running   0          55s
reviews-v2-7df89b6599-tgcvg      2/2     Running   0          55s
reviews-v3-66cc677c87-x82hd      2/2     Running   0          54s
```

You can also check that Istio is working with a describe to some pod, inside it you can see the container as istio-proxy:

`kubectl describe <any pod> 
`
```bash
istio-proxy:
    Container ID:  containerd://ec87a1ec38505ea1b3eb158918f7fcf090220e143bf52245f10b397ad38d24e2
    Image:         docker.io/istio/proxyv2:1.14.3
    Image ID:      docker.io/istio/proxyv2@sha256:b7dc502a51251f7e97ae849382177ebd0ab19668e40f49d6bab5048debfefd48
    Port:          15090/TCP
    Host Port:     0/TCP
    Args:
      proxy
      sidecar
      --domain
      $(POD_NAMESPACE).svc.cluster.local
      --proxyLogLevel=warning
      --proxyComponentLogLevel=misc:error
      --log_output_level=default:info
      --concurrency
```


## Determining the ingress IP and ports

To be able to access externally the Cluster-IP will be associated to a Load Balancer (In GCP), it is checked here that the service is of Type = LoadBalancer

```bash
kubectl get svc istio-ingressgateway -n istio-system**

NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.7.245.63   34.134.221.81   15021:31527/TCP,80:31766/TCP,443:31665/TCP,31400:30649/TCP,15443:31076/TCP   32m
```

# Configuring and testing Ingress IP and Ports 

```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
```

# Test External IP with port:

`echo $INGRESS_HOST $INGRESS_PORT
`

`Output: <Your Public IP> <You Port normally 80>
`
#Associate Gateway:

`export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
`
###Test with Curl and after in your Browser:

```bash
echo "http://$GATEWAY_URL/productpage"

http://34.134.221.81:80/productpage
```

In your explorer you will have  something like:

[![IMAGE PRODUCTPAGE](IMAGE PRODUCTPAGE "IMAGE PRODUCTPAGE")](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/7e99ce0c-7afa-4182-8230-d073240b4c23/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220905%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220905T210141Z&X-Amz-Expires=86400&X-Amz-Signature=97f9d8001068eb247d8a9511c81fb08c0b3875421854adc7a25d6909834a7624&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22&x-id=GetObject "IMAGE PRODUCTPAGE")

## Adding Telemetry with Istio

```bash
ls -lrt samples/addons
total 560
-rw-r--r--  1 diegocalpa  staff   14114 Jul 28 18:30 prometheus.yaml
-rw-r--r--  1 diegocalpa  staff   11727 Jul 28 18:30 kiali.yaml
-rw-r--r--  1 diegocalpa  staff    2533 Jul 28 18:30 jaeger.yaml
-rw-r--r--  1 diegocalpa  staff  245578 Jul 28 18:30 grafana.yaml
drwxr-xr-x  6 diegocalpa  staff     192 Jul 28 18:30 extras
-rw-r--r--  1 diegocalpa  staff    5194 Jul 28 18:30 README.md

kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system
```

The last command will install all availables telemetry tools for Istio, you can check verifying the services:

```bash
k get svc -n istio-system
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                                                      AGE
grafana                ClusterIP      10.7.251.35    <none>          3000/TCP                                                                     80s
istio-egressgateway    ClusterIP      10.7.242.241   <none>          80/TCP,443/TCP                                                               45m
istio-ingressgateway   LoadBalancer   10.7.245.63    34.134.221.81   15021:31527/TCP,80:31766/TCP,443:31665/TCP,31400:30649/TCP,15443:31076/TCP   45m
istiod                 ClusterIP      10.7.243.223   <none>          15010/TCP,15012/TCP,443/TCP,15014/TCP                                        46m
jaeger-collector       ClusterIP      10.7.242.23    <none>          14268/TCP,14250/TCP,9411/TCP                                                 73s
kiali                  ClusterIP      10.7.244.68    <none>          20001/TCP,9090/TCP                                                           68s
prometheus             ClusterIP      10.7.249.251   <none>          9090/TCP                                                                     65s
tracing                ClusterIP      10.7.243.74    <none>          80/TCP,16685/TCP                                                             74s
zipkin                 ClusterIP      10.7.254.57    <none>          9411/TCP

```

Is possible execute an automatic port-forwarding in order to have access to Grafana, Kiali, Prometheus.

```bash
inside Istio/bin path:
istioctl dashboard <tool>
```

## Grafana:

istioctl dashboard grafana
http://localhost:3000


## Kiali

bin ./istioctl dashboard kiali
http://localhost:20001/kiali (user: admin pass: admin )


Thats all, enjoy it.

Diego Coral.
