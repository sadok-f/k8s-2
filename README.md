# k8s

## Tutorial Objective:
- [x] Create a Kubernetes cluster
- [x] Install Helm
- [x] Deploy spinnaker
- [ ] Deploy istio
- [ ] Demonstrate the deployment of an example application (micro-services) with automatic canary analysis
- [ ] Configuration of request routing in Istio.

## Create a Kubernetes cluster
Following the tutorial from @kelseyhightower [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way).
In order to setup GCE as cloud provider for the final cluster, this changes need to be done:
- In the 3rd step (Provisioning Compute Resources), change the instance type for the workers from `n1-standard-1` to `n1-standard-2` 
- Add `--cloud-provider=gce` for the services  kube-apiserver (/etc/systemd/system/kube-apiserver.service),kube-controller-manager (/etc/systemd/system/kube-controller-manager.service) and kubelet (/etc/systemd/system/kubelet.service)
- Add `--hostname-override=worker-X` for each kubelet service in the workers (replace X by the number of the worker such as 0, 1, 2)
- Add a standard `StorageClass` ressource to the cluster with `provisioner: kubernetes.io/gce-pd`:
```
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: EnsureExists
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
EOF
```
By the end of the tutorial we'll have the following resources:
```
$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
```
```
kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
worker-0   Ready    <none>   3d15h   v1.12.0
worker-1   Ready    <none>   3d15h   v1.12.0
worker-2   Ready    <none>   3d15h   v1.12.0
```

## Install Helm

Install Helm in local MacOs
```
brew install kubernetes-helm
```

Create the necessary roles for Helm
```
kubectl create clusterrolebinding user-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)

kubectl create serviceaccount tiller --namespace kube-system 

kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

kubectl create clusterrolebinding --clusterrole=cluster-admin	--serviceaccount=default:default spinnaker-admin
```

Initialize Helm
```
helm init --service-account=tiller --upgrade 

helm repo update
```

## Install Spinnaker

```
export PROJECT=$(gcloud info --format='value(config.project)')

export BUCKET=$PROJECT-spinnaker-config

gsutil mb -c regional -l europe-west3 gs://$BUCKET

export SA_JSON=$(cat spinnaker-sa.json)
export PROJECT=$(gcloud info --format='value(config.project)')
export BUCKET=$PROJECT-spinnaker-config
cat > spinnaker-config.yaml <<EOF
gcs:
  enabled: true
  bucket: $BUCKET
  project: $PROJECT
  jsonKey: '$SA_JSON'

dockerRegistries:
- name: gcr
  address: https://gcr.io
  username: _json_key
  password: '$SA_JSON'
  email: 1234@5678.com

# Disable minio as the default storage backend
minio:
  enabled: false

# Configure Spinnaker to enable GCP services
halyard:
  spinnakerVersion: 1.12.5
  image:
    repository: gcr.io/spinnaker-marketplace/halyard
    tag: 1.16.0
  additionalScripts:
    create: true
    data:
      enable_gcs_artifacts.sh: |-
        \$HAL_COMMAND config artifact gcs account add gcs-$PROJECT --json-path /opt/gcs/key.json
        \$HAL_COMMAND config artifact gcs enable
      enable_pubsub_triggers.sh: |-
        \$HAL_COMMAND config pubsub google enable
        \$HAL_COMMAND config pubsub google subscription add gcr-triggers \
          --subscription-name gcr-triggers \
          --json-path /opt/gcs/key.json \
          --project $PROJECT \
          --message-format GCR
EOF


helm install -n spinnaker stable/spinnaker -f spinnaker-config.yaml --timeout 600 --version 1.1.6 --wait
```


## Deploy istio
*WIP*
