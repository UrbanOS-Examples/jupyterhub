# jupyterhub

Jupyterhub for the Smart City Operating System

## Setting up OAuth & Jenkins

- Log into Github as SmartColumbusOS admin user
- Go to Organization -> SmartColumbusOS -> Settings
  - https://developer.github.com/apps/managing-oauth-apps/modifying-an-oauth-app/
- Register new application with callback url of `https://jupyterhub.${environment}.internal.smartcolumbusos.com/hub/oauth_callback`
- Generate a secret token
- Add `jupyterhub-${environment}` secret text to Jenkins credentials
  - These are passed as ENV arguments to a docker run command
  - ex: `-e SECRET_TOKEN=somestringwithgoodentropy -e CLIENT_ID=${get me from github} -e CLIENT_SECRET=${get me from github}`

## Restoring notebook volumes

Each user on JupyterHub will have a personal EBS volume created in AWS for them that will persist through anything short of its kubernetes cluster being torn down completely. In cases where a cluster is deleted and a new one is created the volumes still exist, but kubernetes no longer knows about them. To restore the data in the volumes you must re-attach them to kubernetes so the containers can find them.

Diagram, normal operation:

```
+-----------------------------------+            +---------------------------------+             +----------------------------------+             +-----------------------------------------+
|                                   |            |                                 |             |                                  |             |                                         |
|                                   |            |                                 |             |                                  |             |                                         |
|                                   |            |                                 |             |                                  |             |                                         |
|                                   |            |                                 |             |                                  |             |                                         |
|                                   |            |     Persistent Volume Claim     |             |       Persistent Volume X        |             |                                         |
|      Notebook: notebook-Y         +----------> |     Use: Persistent Volume X    +-----------> |       Use: aws://volume-Y        +-----------> |          AWS Volume: volume-Y           |
|                                   |            |                                 |             |                                  |             |                                         |
|                                   |            |                                 |             |                                  |             |                                         |
|                                   |            |                                 |             |                                  |             |                                         |
|                                   |            |                                 |             |                                  |             |                                         |
+-----------------------------------+            +---------------------------------+             +----------------------------------+             +-----------------------------------------+
```

Restoration steps:

- If the user has already started a server/notebook on the new cluster then you will need to delete the PVC for that user. Otherwise, you can skip this step.

```
> kubectl delete pod jupyter-<username> -n jupyterhub
> kubectl delete pvc claim-<username> -n jupyterhub
```

- Each user volume should show up in EC2 EBS with tags showing that they are for jupyter. You can filter the volumes using their `<username>` in the GUI. Example of tags:

```
kubernetes.io/created-for/pv/name: pvc-<uuid>
kubernetes.io/created-for/pvc/name: claim-<username>
kubernetes.io/created-for/pvc/namespace: jupyterhub
```

- Create a PV pointed at the desired volume (you can re-use the `pv/name` tag for the `name` field or specify `generateName: prefix-` field to generate a new one):

```
> cat pv-<username>.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    kubernetes.io/createdby: aws-ebs-dynamic-provisioner
    pv.kubernetes.io/bound-by-controller: "yes"
    pv.kubernetes.io/provisioned-by: kubernetes.io/aws-ebs
  finalizers:
  - kubernetes.io/pv-protection
  name: pvc-<uuid>
  labels:
    failure-domain.beta.kubernetes.io/region: us-west-2
    failure-domain.beta.kubernetes.io/zone: us-west-2a
spec:
  accessModes:
  - ReadWriteOnce
  awsElasticBlockStore:
    fsType: ext4
    volumeID: aws://us-west-2a/<volume id of the desired volume>
  capacity:
    storage: 10Gi
  mountOptions:
  - debug
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard-ssd

> kubectl create pv -f pv-<username>.yaml -n jupyterhub
```

- Create a PVC pointed at the previously created PV and with the correct `claim-` name:

```
> cat pvc-<username>.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    hub.jupyter.org/username: <username>
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/aws-ebs
  finalizers:
  - kubernetes.io/pvc-protection
  labels:
    app: jupyterhub
    heritage: jupyterhub
  name: claim-<username>
  namespace: jupyterhub
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard-ssd
  volumeName: <pvc name from previous step>
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  phase: Bound
> kubectl create pvc -f pvc-<username>.yaml -n jupyterhub
```

- Now the user can start their server/notebook and their old notebooks should be available.

### Backing Up

In the event we need to migrate the Kubernetes cluster from one region to another,
or otherwise perform an action that would cause us to destroy the k8s cluster,
we need to backup all of the Kubernetes PersistentVolumes and PersistentVolumeClaims.

```bash
kubectl get pv -n jupyterhub -o yaml > jupyter-persistent-volume.bak
kubectl get pvc -n jupyterhub -o yaml > jupyter-persistent-volume-claim.bak
```

These will need to be re-applied to the new kubernetes cluster in order to reattach the claims.

### Running Jupyterhub in Sandbox workspace

Running these scripts verbatum may not be 100% accurate but it should hopefully be a good guide. You will need to update the callback url to match your new environment you'll be standing up in sandbox. A github adminstrator will need to change the value here:
https://github.com/organizations/SmartColumbusOS/settings/applications/867939

```
export AWS_PROFILE=<SANDBOX_PROFILE>
export CARD_NUMBER=smrt-455
cd ~/code/common/env
tf-init --sandbox -w $CARD_NUMBER
\#Doing apply here and not a plan because it asks you before continuing
tf apply --var-file=variables/sandbox.tfvars
export KUBECONFIG=~/code/common/env/kubeconfig_streaming-kube-$CARD_NUMBER
kubectl apply -f ~/code/common/env/k8s/tiller-role/01-tiller-admin.yaml
kubectl apply -f ~/code/common/env/k8s/persistent-storage/01-storage-class.yaml
```

Substitute $ values in ~/code/common/env/k8s/alb-ingress-controller/02-deployment.yaml can get them from terraform output

```
kubectl apply -f ~/code/common/env/k8s/alb-ingress-controller/
```

```
cd ~/code/jupyterhub
docker build -t $CARD_NUMBER .
DOCKER_USER=$(aws ecr get-login --no-include-email --region us-east-2 | cut -d ' ' -f4)
DOCKER_PASSWD=$(aws ecr get-login --no-include-email --region us-east-2 | cut -d ' ' -f6)
SERVER=$(aws ecr get-login --no-include-email --region us-east-2 | cut -d ' ' -f7)
SANDBOX_CLIENT_ID=<client ID here>
SANDBOX_CLIENT_SECRET=<client secret here>
docker run -it -e SECRET_TOKEN=<secret token here> -e CLIENT_ID=$SANDBOX_CLIENT_ID -e CLIENT_SECRET=$SANDBOX_CLIENT_SECRET -e LEAFLET_NOTEBOOK_TAG=latest -e CALLBACK_URL=https://jupyter.$CARD_NUMBER.sandbox.internal.smartcolumbusos.com -e DOCKER_USER=$DOCKER_USER -e DOCKER_PASSWD=$DOCKER_PASSWD -e SERVER=$SERVER -e AWS_PROFILE=$AWS_PROFILE -v ~/code/common/env/kubeconfig_streaming-kube-smrt-455:/root/.kube/config -v ~/.aws/credentials:/root/.aws/credentials $CARD_NUMBER bash
```

Once in the docker container execute the `run.sh` script.

Substitute $ values in the two files below, can get them from terraform output

```
kubectl apply -f ~/code/jupyterhub/k8s/deployments/
```
