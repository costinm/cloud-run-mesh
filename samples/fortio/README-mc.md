# Sample with multiple containers


## Cluster setup 

The cluster setup is similar to 'golden image', the project Makefile includes
targets to deploy istiod and the Mesh Connector. The Mesh Connector is not 
used in this setup except for creating a config map that simplifies the setup,
will likely be removed as a requirement.


## Workload deployment 

The examples will create 2 services, fortio-vpc-mc1 and fortio-vpc-mc2.

There are 2 steps - configuring K8S, to be aware of the services and enable
auto-registration, and deploying the app in cloudrun.

```shell
export PROJECT_ID=wlhe-cr
export WORKLOAD_NAMESPACE=fortio
export REGION=us-central1

# K8S cluster holding istio configs ( can be autopilot )
export CONFIG_PROJECT_ID=wlhe-cr
export CLUSTER_LOCATION=us-central1-c
export CLUSTER_NAME=istio


export CLOUDRUN_SERVICE=fortio-vpc-mc1

cat k8s-registration.yaml |  envsubst | kubectl apply -f -

cat cloudrun-workload-service.yaml | envsubst > /tmp/${CLOUDRUN_SERVICE}.yaml

gcloud alpha run services replace /tmp/${CLOUDRUN_SERVICE}.yaml --project ${PROJECT_ID}

# Same thing, for the second service
export CLOUDRUN_SERVICE=fortio-vpc-mc2

cat k8s-registration.yaml |  envsubst | kubectl apply -f -

cat cloudrun-workload-service.yaml | envsubst > /tmp/${CLOUDRUN_SERVICE}.yaml

gcloud alpha run services replace /tmp/${CLOUDRUN_SERVICE}.yaml --project ${PROJECT_ID}


```

Once the command completes, both workloads should be working.

It is possible to deploy additional services and deployments in K8S or VMs.
The K8S Service created in the first step will have the name ${CLOUDRUN_SERVICE}
and select any workload with the label "app: ${CLOUDRUN_SERVICE}" - so any
request from the K8S cluster or from a CloudRun instance will be balanced across
workloads running in all environments. 

For this example we are only running workloads in CloudRun, so requests will be made
from the first service to the second. 

To verify the workloads are registered or to debug, run:

```
kubectl -n ${WORKLOAD_NAMESPACE} get workloadentry

```

The result should show entries with the name of the service followed by "-" and an IP address.
That is the IP address of the instance - depending on auto-scaling it may be one or more IPs.
When the workload scales down, the entry will be garbage collected and removed.

The sidecar image includes a ssh server on port 15022 - for debugging you can 
connect to it, but it must be from the VPC, for example using a VM or Pod running
on the same network.

Note: currently minInstance=1 is required for workloads.

### Accessing the workloads

The 2 workloads can be accessed in multiple ways:

1. From the mesh: http or ssh calls from any Pod or VM in the VPC
2. From the internet: in CloudConsole you can change the access setting to allow 
unauthenticated access, or use JWT tokens to access the service
3. From Istio Gateways - the services act like any K8S Service
4. From Istio Gateways running in CloudRun - next section.

## Gateway on CloudRun deployment

Istio Gateways are normally deployed in cluster. Users need to use CertManager and few 
complex steps to get basic TLS working, and the gateway deployment is usually shared
per cluster.

It is also possible now to deploy a Istio Gateway as a CloudRun service. The gateway
will automatically get a certificate and can scale to zero when not in use. It is possible
to deploy the gateway as either 'per cluster' or for a single namespace. 

Currently only port 80 and 443 are supported.

To deploy:

1. Configure the Istio Gateway - the sample includes VirtualServices for the 
workload created previously.
```
# Configure the Istio routes and Gateway object
# TODO: switch to K8S Gateway API 
kubect apply -f gateway.yaml

```

2. Deploy the Gateway running in CloudRun.
```
export GATEWAY_NAME=fortiogw

gcloud alpha run deploy ${GATEWAY_NAME} \
		  --execution-environment=gen2 \
		  --platform managed --project ${PROJECT_ID} --region ${REGION} \
		  --service-account=${CLOUDRUN_SERVICE_ACCOUNT} \
          --allow-unauthenticated  \
         \
         --port 8080 \
         \
         --concurrency 10 --timeout 900 --cpu 1 --memory 1G \
         --min-instances 0 --max-instances 3 \
         --network=default --subnet=default \
         \
		--image ${KRUN_IMAGE} \
		\
		--set-env-vars="ISTIO_META_INTERCEPTION_MODE=NONE" \
		--set-env-vars="TRUST_DOMAIN=cluster.local" \
		--set-env-vars="GATEWAY_NAME=${GATEWAY_NAME}" \
		--set-env-vars="MESH=//container.googleapis.com/projects/${CONFIG_PROJECT_ID}/locations/${CLUSTER_LOCATION}/clusters/${CLUSTER_NAME}" \
		--set-env-vars="DEPLOY=$(shell date +%y%m%d-%H%M)"


```

This example creates a namespace-owned gateway. Unlike regular Istio
gateways, there is no need to provision DNS and certificates, and no k8s-stored 
Secrets used for certs. 

We can access the 2 services using the gateway as:
[$CLOUDRUN_ADDRESS/fortio-vpc-mc1/fortio/](https://fortiogw-icq63pqnqq-uc.a.run.app/fortio-vpc-mc1/fortio/)
and [$CLOUDRUN_ADDRESS/fortio-vpc-mc2/fortio/](https://fortiogw-icq63pqnqq-uc.a.run.app/fortio-vpc-mc2/fortio/), based on the 
2 routes configured in the fortio-gw VirtualService associated with the fortiogw Gateway.

From each exposed app, we can connect to either K8S services or other 
cloudrun services using the k8s service name - fortio-vpc-mc1 or 
fortio-vpc-mc1.fortio, fortio-vpc-mc1.fortio.svc. 


## Mixed services

The k8s service in k8s-registration.yaml has a selector with "app: service-name".

That means any pod, VM or Cloudrun instance configure with that label will
be automatically added to the service. It is possible to run a Deployment
or a VM with the same label, with traffic balanced across all environments.

This is convenient for migrating workloads between environments. 

## Auto-registration

The WorkloadGroup config enables auto-registration. You can check the CloudRun instances
using 

```shell

kubectl -n fortio get we
...
fortio-vpc-mc1-10.128.0.128   20h    10.128.0.128

```

For debugging, using a VM or Pod you can connect to the CR instance using


```shell
ssh -p 15022  10.128.0.128

...
iptables-save -c 
tcpdump
...

```
