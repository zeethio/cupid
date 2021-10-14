# cupid

<p align="center">
<img src="doc/cupid.png" />
</p>

 <br/><br/>
### Venus service cluster deployment with kubernetes/helm charts (Under Development). <br/><em>Venus services provide shared infrastructure for Filecoin storage providers, a form of storage pooling that reduces cost of operation. Visit https://venus.filecoin.io/ for the official documentaion of the Venus project</em>.

This repo contains experimental Kubernetes/Helm charts to deploy the venus services. It may be helpful for the storage providers exiting from the venus incubation program to setup their own service maintain it easily.

 <br/><br/>
<p align="center">
<img src="doc/deployment.svg" />
</p>
 <br/><br/>

Note: The information presented in this repo is informal and the purpose is to help undertand the organization of kubernetes/Helm charts used for deployment of venus services. 
For offical documentation of venus services and architecture vist:  
https://venus.filecoin.io/guide/
## Prerequisites:

- kubernetes (v 1.22 or above) cluster with nodes suitable for running venus(requires > 2TiB of storage) and other modules. Refer the venus documentation of recommended HW for running the Venus services (https://venus.filecoin.io/master/Chain_service_construction.html#chain-services)

- Helm v3
## Repo organization:
- cupid/cupid repo is an umbrella Helm chart that contains a seperate subchart for each venus module. For example cupid/cupid/charts/venus contains Helm sub chart for installing venus module.
- The docker files for various venus submodules are located in subfolders with names corresponding to the git repo names i.e. cupid/venus, cupid/venus-auth, cupid/venus-messager, cupid/venus-gateway and cupid/venus-miner. Modify the git repo checkout command in the Docker file to the required release tag of the module. Currently the Docker file includes build instructions from the venus deployment guide (https://venus.filecoin.io/guide/How-To-Deploy-MingPool.html). We need to remove build intermediaries to reduce image size.
- helm charts are located in the cupid/charts folder as subcharts. Currently the charts refer to Docker hub repo "zeethio". You may to change to your own repo.
- cupid/storage folder contains PersistentVolume allocated for venus as local disc folder and for venus-auth as nfs mount.
- cupid/secrets contain template for specifying shared-admin token 
- There is some skeletal documentation for each module in the subfolders along with Dockerfile. These docs are created to gain insight into the communication across modules. For accurate interpretation of API calls refer to Venus official documentation and source code.
## Deployment Notes:
- Currently the venus modules are deployed as StatefulSets and exposed through NodePort service. In future it will be changed to Deployments exposed through clusterIP and load balancing for scalability and reliability.

### Preparation
- Identify cluster node for venus, allocate local disk folder. Create nfs mount for venus-auth. Create the peristant volumes before running helm charts

```
kubectl create -f storage/pv-cupid-venus-1.yaml
kubectl create -f pv-cupid-auth.yaml
```
Install Helm chart for venus-auth service
```
helm install cupid-auth cupid/charts/auth
```

Generate shared admin token

```
kubectl exec --tty --stdin auth-0  -- /app/venus-auth/venus-auth --repo /data/repo token gen cupid --perm admin
```
Copy the template file secrets/shared-token-template.yaml to a file, for example shared-token.yaml. Update the adminToken in this file with the token generated from the above command.

Deploy the secrets
```
kubectl create -f shared-token.yaml
```

Setup the global variable that defines service endpoints.
```
kubectl create -f globals/config-map.yaml
```

Install Helm charts for venus node service.
The chart value is by default start from a snaphot of filecoin blockchain. This will accelerate syncing of the node. The snapshot file can be from mounted local disc volume of venus pod or from url. setting it to null (i.e. --set arguments.snapshot=null supplied during helm install of venus chart) starts the blockchain state from the previous state.
Set the value in cupid/charts/venus/values.yaml
arguments:
  snapshot: /data/minimal_finality_stateroots_latest.car

```
helm install cupid-venus cupid/charts/venus
```

Install Helm charts for all the remaining venus services
```
helm install cupid-gateway cupid/charts/gateway
helm install cupid-messager cupid/charts/messager
helm install cupid-miner cupid/charts/miner
```

## Configuring venus-sealer, venus-wallet to use the Venus Chain Serivces
An account name and miner-id are used to generate authentication tokens to give access to the Venus Chain Services. The tokens are generated using venus-auth and configured in the venus-sealer and venus-wallet configuration files. We give an example configuration and steps below: 
Example user account name: zeethio,  miner id: t01192664
```
kubectl exec --tty --stdin auth-0  -- /app/venus-auth/venus-auth --repo /data/repo user add --name zeethio --miner=t01192664
```
```
kubectl exec --tty --stdin auth-0  -- /app/venus-auth/venus-auth --repo /data/repo token gen zeethio --perm write
```
Use the token generated above to configure venus-sealer and venus-wallet.

Use the notePorts and node addresses exposed from the cluster to access the services.

Look at the following venus incubation exit guideline:
https://venus.filecoin.io/master/Incubation_exit_guide.html#incubation-exit-guide

Example sealer configuration for venus services hosted at cupid.zeeth.io
venus node pod/service port 3453 mapped to Nodeport 30002.
venus-messager pod/service port 39812 mapped to Nodeport 30004.
venus-gateway  pod/service port 45132 mapped to Nodeport 30003.
```
[Node]
  Url = "/dns/cupid.zeeth.io/tcp/30002"
  Token = <AUTH_TOKEN_FOR_ACCOUNT_NAME>

[Messager]
  Url = /ip4/cupid.zeeth.io/tcp/30004
  Token = <AUTH_TOKEN_FOR_ACCOUNT_NAME>

[RegisterProof]
  Urls = ["/dns/cupid.zeeth.io/tcp/30003"]
  Token = <AUTH_TOKEN_FOR_ACCOUNT_NAME>

```

Example wallet configuration for venus services hosted at cupid.zeeth.io.  
venus-gateway  pod/service port 45132 mapped to Nodeport 30003.
```
[APIRegisterHub]
RegisterAPI = ["/dns/cupid.zeeth.io/tcp/30003"]
Token = "<AUTH_TOKEN_FOR_ACCOUNT_NAME>"
SupportAccounts = ["zeethio"]
```