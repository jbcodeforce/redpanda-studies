# Kubernetes

## OpenShift Deploy:

* See [deploy app with Helm 3](https://www.openshift.com/blog/openshift-4-3-deploy-applications-with-helm-3):

  ```shell
  curl -L https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-darwin-amd64 -o /usr/local/bin/helm
  chmod +x /usr/local/bin/helm
  helm version
  ```

* Create redpanda project: `oc new-project redpanda`
* Get the [Helm Chart](https://github.com/vectorizedio/helm-charts) and modify some values in the values.yaml

  ```shell
  git clone git@github.com:vectorizedio/helm-charts.git
  ```

* update values.yaml to modify the security context and specify the storage class

  ```yaml
  serviceAccount:
    create: true
    annotations: {}
    name: "redpanda-sa"
    
  podSecurityContext: {}
  securityContext: 
      readOnlyRootFilesystem: true
      runAsNonRoot: true
  storage:   
    storageClass: "ibmc-block-gold"
  ```

* Install

  ```shell
  helm install --namespace redpanda redpanda ./redpanda/
  ```

  The output looks like:

  ```shell
  NAME: redpanda
  LAST DEPLOYED: Mon Feb  8 16:49:18 2021
  NAMESPACE: redpanda
  STATUS: deployed
  REVISION: 1
  NOTES:
  Congratulations on installing redpanda!
  ```

* The pods will rollout in a few seconds. To check the status:

  ```sh
    kubectl -n redpanda rollout status -w statefulset/redpanda
  ```

* Try some sample commands, like creating a topic called topic1:

  ```shell
  oc  run -ti --rm --restart=Never \
    --image vectorized/redpanda:latest \
    rpk -- --brokers=redpanda-bootstrap:9092 api topic create topic1
  ```

* To get the api status:
 
  ```shell
  kubectl -n redpanda run -ti --rm --restart=Never \
    --image vectorized/redpanda:latest \
    rpk -- --brokers=redpanda-bootstrap:9092 api status

  ```

* Get services: `oc get svc` and then expose the bootstrap as route: `oc expose svc redpanda-bootstrap`

