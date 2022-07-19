# PowerScale-CSI-Helm

## Youtube video:
- https://youtu.be/UcAD8upATek

## Login to the Isilon web interface
- Navigate to "File System" --> "File system explorer"
- Create a new directory un the /ifs/data path called csi (i.e. /ifs/data/csi)
- Assign the root user from the File:System provider
- Navigate to "Protocols" --> "UNIX sharing (NFS)" --> "Global settings"
- Check the "Enable NFSv4" checkbox and save changes
- Go to the "Export settings" tab
- Change the "Root user mapping" setting to "Use custom" and "Map root users to a specified user"
- Click the 'browse' button next to user and select the "File:System" provider in the drop down list
- Click 'search' and select the root user

## Install helm
- curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

## Clone the github repo for the CSI driver
- git clone -b v2.2.0 https://github.com/dell/csi-powerscale.git

## Create a new namespace
- kubectl create namespace isilon

## Update the secret.yaml file with the clustername, username, password, endpoint, skipvalidation (true), isipath for Isilon
- cd csi-powerscale
- cp samples/secret/secret.yaml isilon-creds.yaml
- vi isilon-creds.yaml

## Use these two files to create the secrets
- kubectl create secret generic isilon-creds -n isilon --from-file=config=isilon-creds.yaml -o yaml --dry-run=client | kubectl apply -f -
- kubectl create -f samples/secret/empty-secret.yaml

## Update the settings file to replace "isiAuthType: 0" with "isiAuthType: 1"
- cp helm/csi-isilon/values.yaml isilon-values.yaml
- vi isilon-values.yaml

############################################################
## Run the installer script
############################################################

- dell-csi-helm-installer/csi-install.sh --namespace isilon --values isilon-values.yaml

## Verify the driver was installed successfully with all components required
- dell-csi-helm-installer/verify.sh --namespace isilon --values isilon-values.yaml

## When completed, without any errors, there should be 5 isilon controller pods and 2 x 2 isilon node pods running

- kubectl get pods -n isilon

```
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
isilon               isilon-controller-c7c5988b6-4kg92            5/5     Running   0          33m
isilon               isilon-controller-c7c5988b6-whgcz            5/5     Running   0          33m
isilon               isilon-node-dq6rc                            2/2     Running   0          33m
isilon               isilon-node-f8jhb                            2/2     Running   0          33m
```
## Create a storage class
- kubectl create -f samples/storageclass/isilon.yaml

## Create a volume snapshot class
- kubectl create -f samples/volumesnapshotclass/isilon-volumesnapshotclass-v1.yaml

#############################################
## Test the CSI driver
#############################################

- kubectl create namespace test
- kubectl config set-context --current --namespace=test
- cd ~
- curl -LO https://github.com/theocrithary/PowerScale-CSI-Helm/raw/main/pvc.yaml
- kubectl create -f pvc.yaml
- curl -LO https://github.com/theocrithary/PowerScale-CSI-Helm/raw/main/test_pod.yaml
- kubectl create -f test_pod.yaml
- curl -LO https://github.com/theocrithary/PowerScale-CSI-Helm/raw/main/snapshot-of-pvol0.yaml
- kubectl create -f snapshot-of-pvol0.yaml

## Check that all items are created successfully
- kubectl get persistentvolumes
- kubectl get volumesnapshot
- kubectl describe storageclass isilon
- kubectl describe pod isilontestpod1

## Log into the pod and test the PV for read/write access
- kubectl exec -it isilontestpod1 -- bash
- mount | grep data0
- cd /data0
- touch test

## Clean up and remove all test items
- kubectl delete volumesnapshot snapshot-of-pvol0
- kubectl delete pod isilontestpod1
- kubectl delete pvc pvol0
- kubectl delete ns test
