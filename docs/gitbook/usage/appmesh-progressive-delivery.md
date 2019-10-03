# App Mesh Canary Deployments

This guide shows you how to use App Mesh and Flagger to automate canary deployments. 
You'll need an EKS cluster configured with App Mesh, you can find the install guide 
[here](https://docs.flagger.app/install/flagger-install-on-eks-appmesh).

### Bootstrap

Flagger takes a Kubernetes deployment and optionally a horizontal pod autoscaler (HPA), 
then creates a series of objects (Kubernetes deployments, ClusterIP services, App Mesh virtual nodes and services). 
These objects expose the application on the mesh and drive the canary analysis and promotion.
The only App Mesh object you need to create by yourself is the mesh resource.

Create a mesh called `global`:

```bash
export REPO=https://raw.githubusercontent.com/weaveworks/flagger/master

kubectl apply -f ${REPO}/artifacts/appmesh/global-mesh.yaml
```

Create a test namespace with App Mesh sidecar injection enabled:

```bash
kubectl apply -f ${REPO}/artifacts/namespaces/test.yaml
```

Create a deployment and a horizontal pod autoscaler:

```bash
kubectl apply -f ${REPO}/artifacts/appmesh/deployment.yaml
kubectl apply -f ${REPO}/artifacts/appmesh/hpa.yaml
```

Deploy the load testing service to generate traffic during the canary analysis:

```bash
helm upgrade -i flagger-loadtester flagger/loadtester \
--namespace=test \
--set meshName=global \
--set "backends[0]=podinfo.test" \
--set "backends[1]=podinfo-canary.test"
```

Create a canary custom resource:

```yaml
apiVersion: flagger.app/v1alpha3
kind: Canary
metadata:
  name: podinfo
  namespace: test
spec:
  # deployment reference
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  # the maximum time in seconds for the canary deployment
  # to make progress before it is rollback (default 600s)
  progressDeadlineSeconds: 60
  # HPA reference (optional)
  autoscalerRef:
    apiVersion: autoscaling/v2beta1
    kind: HorizontalPodAutoscaler
    name: podinfo
  service:
    # container port
    port: 9898
    # App Mesh reference
    meshName: global
    # App Mesh egress (optional) 
    backends:
      - backend.test
  # define the canary analysis timing and KPIs
  canaryAnalysis:
    # schedule interval (default 60s)
    interval: 10s
    # max number of failed metric checks before rollback
    threshold: 10
    # max traffic percentage routed to canary
    # percentage (0-100)
    maxWeight: 50
    # canary increment step
    # percentage (0-100)
    stepWeight: 5
    # App Mesh Prometheus checks
    metrics:
    - name: request-success-rate
      # minimum req success rate (non 5xx responses)
      # percentage (0-100)
      threshold: 99
      interval: 1m
    - name: request-duration
      # maximum req duration P99
      # milliseconds
      threshold: 500
      interval: 30s
    # testing (optional)
    webhooks:
      - name: acceptance-test
        type: pre-rollout
        url: http://flagger-loadtester.test/
        timeout: 30s
        metadata:
          type: bash
          cmd: "curl -sd 'test' http://podinfo-canary.test:9898/token | grep token"
      - name: load-test
        url: http://flagger-loadtester.test/
        timeout: 5s
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://podinfo.test:9898/"
```

Save the above resource as podinfo-canary.yaml and then apply it:

```bash
kubectl apply -f ./podinfo-canary.yaml
```

After a couple of seconds Flagger will create the canary objects:

```bash
# applied 
deployment.apps/podinfo
horizontalpodautoscaler.autoscaling/podinfo
canary.flagger.app/podinfo

# generated Kubernetes objects
deployment.apps/podinfo-primary
horizontalpodautoscaler.autoscaling/podinfo-primary
service/podinfo
service/podinfo-canary
service/podinfo-primary

# generated App Mesh objects
virtualnode.appmesh.k8s.aws/podinfo
virtualnode.appmesh.k8s.aws/podinfo-canary
virtualnode.appmesh.k8s.aws/podinfo-primary
virtualservice.appmesh.k8s.aws/podinfo.test
virtualservice.appmesh.k8s.aws/podinfo-canary.test
```

After the boostrap, the podinfo deployment will be scaled to zero and the traffic to `podinfo.test` will be routed
to the primary pods. During the canary analysis, the `podinfo-canary.test` address can be used to target directly the canary pods.

The App Mesh specific settings are:

```yaml
  service:
    port: 9898
    meshName: global
    backends:
      - backend1.test
      - backend2.test
```

App Mesh blocks all egress traffic by default. If your application needs to call another service, you have to create an
App Mesh virtual service for it and add the virtual service name to the backend list.

### Setup App Mesh ingress (optional)

In order to expose the podinfo app outside the mesh you'll be using an Envoy ingress and an AWS classic load balancer.
The ingress binds to an internet domain and forwards the calls into the mesh through the App Mesh sidecar.
If podinfo becomes unavailable due to a HPA downscaling or a node restart,
the ingress will retry the calls for a short period of time.

Deploy the ingress and the AWS ELB service:

```bash
kubectl apply -f ${REPO}/artifacts/appmesh/ingress.yaml
```

Find the ingress public address:

```bash
kubectl -n test describe svc/ingress | grep Ingress

LoadBalancer Ingress:     yyy-xx.us-west-2.elb.amazonaws.com
```

Wait for the ELB to become active:

```bash
 watch curl -sS ${INGRESS_URL}
```

Open your browser and navigate to the ingress address to access podinfo UI.

### Automated canary promotion

Trigger a canary deployment by updating the container image:

```bash
kubectl -n test set image deployment/podinfo \
podinfod=stefanprodan/podinfo:3.1.1
```

Flagger detects that the deployment revision changed and starts a new rollout:

```text
kubectl -n test describe canary/podinfo

Status:
  Canary Weight:         0
  Failed Checks:         0
  Phase:                 Succeeded
Events:
 New revision detected! Scaling up podinfo.test
 Waiting for podinfo.test rollout to finish: 0 of 1 updated replicas are available
 Pre-rollout check acceptance-test passed
 Advance podinfo.test canary weight 5
 Advance podinfo.test canary weight 10
 Advance podinfo.test canary weight 15
 Advance podinfo.test canary weight 20
 Advance podinfo.test canary weight 25
 Advance podinfo.test canary weight 30
 Advance podinfo.test canary weight 35
 Advance podinfo.test canary weight 40
 Advance podinfo.test canary weight 45
 Advance podinfo.test canary weight 50
 Copying podinfo.test template spec to podinfo-primary.test
 Waiting for podinfo-primary.test rollout to finish: 1 of 2 updated replicas are available
 Routing all traffic to primary
 Promotion completed! Scaling down podinfo.test
```

When the canary analysis starts, Flagger will call the pre-rollout webhooks before routing traffic to the canary.

**Note** that if you apply new changes to the deployment during the canary analysis, Flagger will restart the analysis.

A canary deployment is triggered by changes in any of the following objects:
* Deployment PodSpec (container image, command, ports, env, resources, etc)
* ConfigMaps mounted as volumes or mapped to environment variables
* Secrets mounted as volumes or mapped to environment variables

During the analysis the canary’s progress can be monitored with Grafana. The App Mesh dashboard URL is 
http://localhost:3000/d/flagger-appmesh/appmesh-canary?refresh=10s&orgId=1&var-namespace=test&var-primary=podinfo-primary&var-canary=podinfo

![App Mesh Canary Dashboard](https://raw.githubusercontent.com/weaveworks/flagger/master/docs/screens/flagger-grafana-appmesh.png)

You can monitor all canaries with:

```bash
watch kubectl get canaries --all-namespaces

NAMESPACE   NAME      STATUS        WEIGHT   LASTTRANSITIONTIME
test        podinfo   Progressing   15       2019-10-02T14:05:07Z
prod        frontend  Succeeded     0        2019-10-02T16:15:07Z
prod        backend   Failed        0        2019-10-02T17:05:07Z
```

If you’ve enabled the Slack notifications, you should receive the following messages:

![Flagger Slack Notifications](https://raw.githubusercontent.com/weaveworks/flagger/master/docs/screens/slack-canary-notifications.png)

### Automated rollback

During the canary analysis you can generate HTTP 500 errors to test if Flagger pauses the rollout.

Trigger a canary deployment:

```bash
kubectl -n test set image deployment/podinfo \
podinfod=stefanprodan/podinfo:3.1.2
```

Exec into the load tester pod with:

```bash
kubectl -n test exec -it deploy/flagger-loadtester bash
```

Generate HTTP 500 errors:

```bash
hey -z 1m -c 5 -q 5 http://podinfo-canary.test:9898/status/500
```

Generate latency:

```bash
watch -n 1 curl http://podinfo-canary.test:9898/delay/1
```

When the number of failed checks reaches the canary analysis threshold, the traffic is routed back to the primary, 
the canary is scaled to zero and the rollout is marked as failed.

```text
kubectl -n test describe canary/podinfo

Status:
  Canary Weight:         0
  Failed Checks:         5
  Phase:                 Failed
Events:
 Starting canary analysis for podinfo.test
 Pre-rollout check acceptance-test passed
 Advance podinfo.test canary weight 5
 Advance podinfo.test canary weight 10
 Advance podinfo.test canary weight 15
 Halt podinfo.test advancement success rate 69.17% < 99%
 Halt podinfo.test advancement success rate 61.39% < 99%
 Halt podinfo.test advancement success rate 55.06% < 99%
 Halt podinfo.test advancement request duration 1.20s > 0.5s
 Halt podinfo.test advancement request duration 1.45s > 0.5s
 Rolling back podinfo.test failed checks threshold reached 5
 Canary failed! Scaling down podinfo.test
```

If you’ve enabled the Slack notifications, you’ll receive a message if the progress deadline is exceeded, 
or if the analysis reached the maximum number of failed checks:

![Flagger Slack Notifications](https://raw.githubusercontent.com/weaveworks/flagger/master/docs/screens/slack-canary-failed.png)