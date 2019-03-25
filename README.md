# peakoperator

`peakoperator` is an [Ansible Operator](https://github.com/operator-framework/operator-sdk/blob/master/doc/proposals/ansible-operator.md) for deploying and managing Peak services on a Kubernetes cluster.

## Building the operator image

The operator image is built using the [operator-sdk](https://github.com/operator-framework/operator-sdk). You will require Docker installed on your host machine.
```
operator-sdk build  quay.io/peak/peak-operator:v0.1.7
docker push quay.io/peak/peak-operator:v0.1.7
```

## Deploying the operator and CRD on OpenShift

As the `system:admin` user, create the CRD and a cluster-role allowing users to query the peak-services API
```
oc create -f deploy/crds/peak_v1alpha1_peakservice_crd.yaml
echo '---
apiVersion: authorization.openshift.io/v1
kind: ClusterRole
metadata:
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
  name: peak-admin-rules
rules:
- apiGroups:
  - services.peak-oss.net
  resources:
  - '*'
  verbs:
  - create
  - update
  - delete
  - get
  - list
  - watch
  - patch' | oc create -f -
```

Login again as a regular user, create a new project, and create the resources required for the operator:
```
oc login -u user1 master.openshift.example.com
oc apply -f deploy
```
Observe the operator pod deploying, and verify that it has started successfully
```
$ oc get pods
NAME                             READY     STATUS    RESTARTS   AGE
peak-operator-66786764db-q5j4r   1/1       Running   0          8s

$ oc logs -f peak-operator-66786764db-q5j4r
{"level":"info","ts":1552279851.1585448,"logger":"cmd","msg":"Go Version: go1.10.3"}
{"level":"info","ts":1552279851.1586218,"logger":"cmd","msg":"Go OS/Arch: linux/amd64"}
{"level":"info","ts":1552279851.158634,"logger":"cmd","msg":"Version of operator-sdk: v0.5.0"}
...
...
...
...
{"level":"info","ts":1552279851.2675447,"logger":"ansible-controller","msg":"Watching resource","Options.Group":"services.peak-oss.net","Options.Version":"v1alpha1","Options.Kind":"PeakService"}
{"level":"info","ts":1552279851.2677855,"logger":"kubebuilder.controller","msg":"Starting EventSource","controller":"peakservice-controller","source":"kind source: services.peak-oss.net/v1alpha1, Kind=PeakService"}
{"level":"info","ts":1552279851.3681989,"logger":"kubebuilder.controller","msg":"Starting Controller","controller":"peakservice-controller"}
{"level":"info","ts":1552279851.4685004,"logger":"kubebuilder.controller","msg":"Starting workers","controller":"peakservice-controller","worker count":1}
```
Create the example PeakService resource
```
oc create -f deploy/crds/services_v1alpha2_peakservice_cr.yaml
```
Verify that the Peak pods begin to deploy. You can follow the operator logs to verify there are no errors during deployment:
```
$ oc get pods
NAME                             READY     STATUS    RESTARTS   AGE
grafana                          0/1       Running   0          16s
influxdb                         1/1       Running   0          26s
peak-operator-66786764db-q5j4r   1/1       Running   0          3m
peakorc                          1/1       Running   0          1m
peakweb                          1/1       Running   0          51s
postgresql                       1/1       Running   0          1m

$ oc logs -f peak-operator-66786764db-q5j4r
{"level":"info","ts":1552280107.1600294,"logger":"logging_event_handler","msg":"[playbook task]","name":"example-peakservice","namespace":"peak1","gvk":"services.peak-oss.net/v1alpha2, Kind=PeakService","event_type":"playbook_on_task_start","job":"3311915848972585513","EventData.Name":"peakservice : Set flask CSRF Secret to present"}
{"level":"info","ts":1552280107.9999423,"logger":"logging_event_handler","msg":"[playbook task]","name":"example-peakservice","namespace":"peak1","gvk":"services.peak-oss.net/v1alpha2, Kind=PeakService","event_type":"playbook_on_task_start","job":"3311915848972585513","EventData.Name":"peakservice : Set peakweb service to present"}
{"level":"info","ts":1552280108.846557,"logger":"logging_event_handler","msg":"[playbook task]","name":"example-peakservice","namespace":"peak1","gvk":"services.peak-oss.net/v1alpha2, Kind=PeakService","event_type":"playbook_on_task_start","job":"3311915848972585513","EventData.Name":"peakservice : Set Peakweb Route to present"}
{"level":"info","ts":1552280109.7446883,"logger":"logging_event_handler","msg":"[playbook task]","name":"example-peakservice","namespace":"peak1","gvk":"services.peak-oss.net/v1alpha2, Kind=PeakService","event_type":"playbook_on_task_start","job":"3311915848972585513","EventData.Name":"peakservice : Set Peakweb Pod to {{ _peak_state }}"}
{"level":"info","ts":1552280110.5215309,"logger":"logging_event_handler","msg":"[playbook task]","name":"example-peakservice","namespace":"peak1","gvk":"services.peak-oss.net/v1alpha2, Kind=PeakService","event_type":"playbook_on_task_start","job":"3311915848972585513","EventData.Name":"peakservice : Verify that the Peakweb Pod is running"}
```
