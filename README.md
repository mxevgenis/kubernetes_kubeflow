# kubernetes_kubeflow

Use the kubeflow-cluster located in the kubeflow-cluster folder to create a kubernetes cluster for kubeflow purposes. The folder contains yaml files which can be executed using ansible in order to create a fully functional kubernetes cluster. The version of kubernetes used in this configuration is v1.21 (this version is also the kubectl and kubeadm version). 
To execute the yaml files with ansible playbooks simple run:
ansible-playbook -i hosts initial.yml
ansible-playbook -i hosts kube-dependencies.yml
ansible-playbook -i hosts master.yml
check the master node using the kubectl get nodes
Wait for the master node to become Ready and then execute
ansible-playbook -i hosts kube-workers.yml
Check the kubernetes nodes by executing kubectl get nodes
When the nodes are Ready check the pods of kubernetes by executing kubectl get pods
Propably you will observe that the coredns is in error state. The root of the problem is the dns nameservers used in the configuration of the pod. To change the nameservers do the following when you are in the master node.
kubectl -n kube-system edit configmaps coredns -o yaml
Then replace the /etc/resolv.conf with 8.8.8.8. Wait a couple of minutes and then you will observe that pods are in ready state.

Kubeflow requires a storage class used for storing data. To enable this feature we chose the rancher provisioner. Execute the following commands to enable the storageclass.
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.23/deploy/local-path-storage.yaml
kubectl -n local-path-storage get pod
kubectl -n local-path-storage get pod -o wide
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get sc

Now we are ready to procceed to the deployment of kubeflow. To this end we will use the manifest folder of this project which based on the 1.3.1 version of kubeflow. Moreover we will install and use the kustomize software and the version 4.5.7. The information regarding the installation of kubeflow based on the manifest files is available on https://github.com/kubeflow/manifests/tree/v1.3.1
To install the kustomize v4.5.7 run the following:
wget https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv4.5.7/kustomize_v4.5.7_linux_arm64.tar.gz
tar xvzf kustomize_v4.5.7_linux_arm64.tar.gz
sudo mv kustomize /usr/local/bin/
sudo chmod +x /usr/local/bin/kustomize
kustomize version

Then go to the manifest file end execute the following:
kustomize build common/cert-manager/cert-manager/base | kubectl apply -f -
kubectl wait --for=condition=ready pod -l 'app in (cert-manager,webhook)' --timeout=180s -n cert-manager
kustomize build common/cert-manager/kubeflow-issuer/base | kubectl apply -f -
kustomize build common/istio-1-16/istio-crds/base | kubectl apply -f -
kustomize build common/istio-1-9/istio-crds/base | kubectl apply -f -
kustomize build common/istio-1-9/istio-namespace/base | kubectl apply -f -
kustomize build common/istio-1-9/istio-install/base | kubectl apply -f -
kustomize build common/dex/overlays/istio | kubectl apply -f -
kustomize build common/oidc-authservice/base | kubectl apply -f -
kustomize build common/knative/knative-serving/base | kubectl apply -f -
kustomize build common/istio-1-9/cluster-local-gateway/base | kubectl apply -f -
kustomize build common/kubeflow-namespace/base | kubectl apply -f -
kustomize build common/kubeflow-roles/base | kubectl apply -f -
kustomize build common/istio-1-9/kubeflow-istio-resources/base | kubectl apply -f -
kustomize build apps/pipeline/upstream/env/platform-agnostic-multi-user | kubectl apply -f -
kustomize build apps/kfserving/upstream/overlays/kubeflow | kubectl apply -f -
kustomize build apps/katib/upstream/installs/katib-with-kubeflow | kubectl apply -f -
kustomize build apps/centraldashboard/upstream/overlays/istio | kubectl apply -f -
kustomize build apps/admission-webhook/upstream/overlays/cert-manager | kubectl apply -f -
kustomize build apps/jupyter/notebook-controller/upstream/overlays/kubeflow | kubectl apply -f -
kustomize build apps/jupyter/jupyter-web-app/upstream/overlays/istio | kubectl apply -f -
kustomize build apps/profiles/upstream/overlays/kubeflow | kubectl apply -f -
kustomize build apps/volumes-web-app/upstream/overlays/istio | kubectl apply -f -
kustomize build apps/tensorboard/tensorboards-web-app/upstream/overlays/istio | kubectl apply -f -
kustomize build apps/tensorboard/tensorboard-controller/upstream/overlays/kubeflow | kubectl apply -f -
kustomize build apps/training-operator/upstream/overlays/kubeflow | kubectl apply -f -
kustomize build apps/tf-training/upstream/overlays/kubeflow/ | kubectl apply -f -
kustomize build apps/mpi-job/upstream/overlays/kubeflow | kubectl apply -f -
kustomize build common/user-namespace/base | kubectl apply -f -

Then to check the status of the pods:
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n auth
kubectl get pods -n knative-eventing
kubectl get pods -n knative-serving
kubectl get pods -n kubeflow
kubectl get pods -n kubeflow-user-example-com

Propably you will observe an error in the form of ImagePullBackoff in the kfserrving container manager and another error in the mpi pod. To solve the issue of the kfserving controller manager, change the image used by docker as it is presented below.
kubectl -n kubeflow  edit pod  kfserving-controller-manager-0
change the image prefix from gcr.io/kfserving to kfserving
