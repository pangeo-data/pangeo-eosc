# How to deploy Pangeo in the infrastructure of the EGI Federation

These are the steps to deploy [Daskhub](https://docs.dask.org/en/stable/deploying-kubernetes-helm.html#helm-install-dask-for-multiple-users)
A [Dask Gateway](https://gateway.dask.org/) enabled [Jupyterhub](https://jupyter.org/hub) using the 
infrastructure of the [EGI Federation](https://www.egi.eu/egi-federation/).

## How to get access

Getting access consists of the following steps:

1. [Sign-up](https://docs.egi.eu/users/aai/check-in/signup/) for an EGI Check-In account.
1. Request to join the `vo.pangeo.eu`
   [Virtual Organisation (VO)](https://confluence.egi.eu/display/EGIG/Virtual+organisation)
   by visiting the [enrollment URL](https://aai.egi.eu/registry/co_petitions/start/coef:386)
   with your EGI Check-In account. The subscription requires approval from the
   VO Managers. For further information, please check the
   [VO ID card](https://operations-portal.egi.eu/vo/view/voname/vo.pangeo.eu).
1. Use [Infrastructure Manager](https://docs.egi.eu/users/compute/orchestration/im/).
   Look at how to add your credentials
   [here](https://docs.egi.eu/users/compute/orchestration/im/dashboard/#cloud-credentials).

## How to deploy Pangeo

A few considerations before we start:

* You need to be a member of the `vo.pangeo.eu` VO. Please
see steps [above](#how-to-get-access).
* We will be using the DaskHub and Pangeo interchangeably along this document.
See the history [here](https://blog.dask.org/2020/08/31/helm_daskhub).

Here is an overview of the steps that we will follow:

1. Configure a DNS name for your Pangeo deployment using the
[Dynamic DNS](https://docs.egi.eu/users/compute/cloud-compute/dynamic-dns/) service.
1. Get credentials from [EGI Check-In](https://docs.egi.eu/users/aai/check-in/) to allow
configuring Jupyterhub authentication using this service, giving access to all members 
of the `vo.pangeo.eu` Virtual Organization to your deployment.
1. Deploy a Kubernetes cluster on top of OpenStack, along with other tools like Grafana.
[Infrastructure Manager Dashboard](https://docs.egi.eu/users/compute/orchestration/im/dashboard/)
(or simply IM Dashboard) will do this all for us automatically.
1. Configure and install the [DaskHub helm chart](https://helm.dask.org/) using Helm.

### Step 1) Get a DNS name

Log into the [Dynamic DNS web GUI portal](https://nsupdate.fedcloud.eu/)
with your EGI Check-In account to configure the DNS host name of your choice.
The web portal is intuitive, and there is also the associated
[documentation](https://docs.egi.eu/users/compute/cloud-compute/dynamic-dns/)
so we will not go into more details here.
Just use the `Add Host` button, and follow the steps.

For the rest of this tutorial, let us consider the `pangeo.vm.fedcloud.eu`
host name.

### Step 2) Get EGI Check-In credentials

You need to follow this step for __every new Pangeo deployment host name__ 
generated, if you want to link it with EGI Check-in (which is recommended).

Please follow the steps on the Check-In documentation for
[Service Providers](https://docs.egi.eu/providers/check-in/sp/). In particular,
the instructions using OpenID Connect as the authentication and authorization
protocol. You need to fill the registration form at 
[EGI Federation Registry](https://aai.egi.eu/federation/).

Check-In runs three separate instances: production (`aai.egi.eu`),
demo (`aai-demo.egi.eu`), and development (`aai-dev.egi.eu`). To quickly test
integration with Check-In, we suggest to configure Pangeo to connect to
the _development_ instance (this way you can self approve the service registration). 
Note that by default the `vo.pangeo.eu` VO only exists
in the production instance of Check-In. Please ask the EGI Check-In team (via
an email to `check-in@egi.eu`) to create the VO in the development instance
of Check-In.

Here are additional details to fill out the registration form:

* `General` tab:
  * `Integration environment`: `Development`
* `Protocol Specific` tab:
  * `Select Protocol`: `OIDC Service`
  * `Client ID`: leave this empty
  * `Application Type`: `Web`
  * `Redirect URI`: `https://pangeo.vm.fedcloud.eu/hub/oauth_callback`. 
     Adapt it to your own host name.
  * `Scope`: select `openid`, `email`, `profile`, `eduperson_entitlement`,
     `eduperson_scoped_affiliation`, and `eduperson_unique_id`.
  * `Grant Types`: `authorization code`
  * `Token Endpoint Authorization Method`: `Client Secret over HTTP Basic`
  * `Client Secret`: leave this empty

Once you submitted the registration form, you'll need to wait for approval, or review the 
request yourself if using the Dev Federation, and then wait for the Deployment to be done.
Then, you'll just need to get the `Client ID` and `Client Secret` that should have been generated. 


### Step 3) Deploy a Kubernetes cluster

Log into the [IM Dashboard](https://im.egi.eu/im-dashboard/)
(if you want to deploy an Elastic cluster, be sure to use operational
instance of IM Dashboard, not `im-dashboard-dev`)
with your EGI Check-In account, and configure your
[credentials](https://docs.egi.eu/users/compute/orchestration/im/dashboard/#cloud-credentials)
with the `vo.pangeo.eu` VO. 

Then click on the
[Kubernetes template](https://im.egi.eu/im-dashboard/configure?selected_tosca=kubernetes.yaml)
and add `Make Kubernetes Virtual Cluster Elastic` and 
`Launch Prometheus + Grafana on top of a Kubernetes Virtual Cluster` as optional features. 
There are also Jupyterhub and Daskhub available templates, but we'll do the deployment of
these services using Helm for now, as Daskhub Tosca template is not yet mature enough.

Out of all the configuration options, please pay special attention to the following:

* `HW Data` tab:
  * Front end node CPUs and Memory: if you want to create a big Kubernetes cluster, set at least 4 CPUs, and 16 GB of memory.
    No user pods will run on this front end node.
  * Worker nodes CPUs and Memory: on CESNET, you can go up to 16 CPUs and 64GB memory.
    All user pods will run on these nodes.
* `Kubernetes Data` tab:
  * `Access Token for the Kubernetes admin user`: Please change it
  * `Version of Kubernetes to install`: we currently only tested with 1.23.11
  * `Flag to install Cert-Manager` must be set to `True`
  * `Email to be used in the Let's Encrypt issuer`: add your preferred email.
  * `DNS name of the public interface of the FE node to generate the certificate`
       must be set to the one configure in [Step 1](#step-1-dns-name).
* `Elastic Data` tab:
  * `Maximum Number of WNs in the cluster`: please set the maximum of node you'll want the 
    Kubernetes cluster to expand to. 
  * `Min Number of free WNs in the cluster`: 0 or 1, WNs without allocated pods. 
    With this option you ensure the number of empty WNs in preparation for a peak 
    workload, and therefore you save some time when growing the cluster.
* `Cloud Provider Selection` tab:
  * Select the [credentials](https://docs.egi.eu/users/compute/orchestration/im/dashboard/#cloud-credentials)
    configured earlier.
  * `Select Site image`: select `ubuntu-focal` or `ubuntu-jammy`.

Please review all configuration options, click `Submit` and wait for the
deployment to finish with status `Configured`. Then click on `Outputs` and
take a note of the IP address assigned. See the
[documentation](https://docs.egi.eu/users/compute/orchestration/im/dashboard/#infrastructures)
for IM Dashboard to learn more.


#### Connect to deployed Kubernetes cluster

In order to be able to issue `kubectl` or `helm` commands described after, you'll need 
to connect to the Kubernetes cluster front end node. 

In order to do that when the deployement is in `Configured` status, you'll need to follow these steps:

1. On IM Dashboard, click on the VM with id 0 from the deployed infrastructure.
1. Download the Credential file (`key.pem` file) on your local computer.
1. Add the key to your ssh agent: 
   `chmod 600 /path/to/key.pem`
   `ssh-add /path/to/key.pem`
1. Then just connect to the VM: `ssh cloudadm@<external_ip_of_frontendnode>`. 
   External IP should be accessible from IM Dashboard Outputs of your ifnrastructure.
1. Launch a bash session using `bash`

From there, you should be able to issue `kubectl` or `helm` commands, for example:

```bash
$ sudo kubectl get pods -A

NAMESPACE              NAME                                                       READY   STATUS      RESTARTS      AGE
cert-manager           cert-manager-5754c77bd9-sw8gj                              1/1     Running     0             36h
cert-manager           cert-manager-cainjector-7bb5c8d6-2xf5w                     1/1     Running     0             36h
cert-manager           cert-manager-webhook-56ccc5ff8f-pj52w                      1/1     Running     0             36h
daskhub                api-daskhub-dask-gateway-655db6fb79-q6626                  1/1     Running     0             11m
daskhub                continuous-image-puller-lkzfc                              1/1     Running     0             11m
daskhub                controller-daskhub-dask-gateway-6d988656cf-66kht           1/1     Running     0             36h
...
```

#### HTTPS

Please go to the [Dynamic DNS web GUI portal](https://nsupdate.fedcloud.eu/)
and update the public IP for the DNS name with the one shown in the `Outputs`
button of the IM Dashboard. Now `Reconfigure` the deployment so `https` is
correctly configured with Let's Encrypt. There is still a missing step that
will be done in the section below (configuring the ingress with correct values
in the Helm values.yaml file).


### Step 4) Deploy the Daskhub helm chart

#### Daskhub with EGI Check-in auth

After you have successfully registered the Pangeo service in the
[EGI Federation Registry](https://aai.egi.eu/federation) and the 
Kubernetes cluster is deployed, below is the
`values.yaml` that you need to deploy a Daskhub with Check-In authentication.

You'll need to replace some values in there:
* `token1` must be replaced with a hash generated using `openssl rand -hex 32` on Linux.
* `token2` must be replaced with another hash generated using `openssl rand -hex 32` on Linux.
* `egi_oauth_client` must be replaced by the `Client ID` created during step 2.
* `egi_oauth_secret` must be replaced by the `Client Secret` created during step 2.
* Don't forget to replace `pangeo.vm.fedcloud.eu` with your DNS name.

You might want also to modify other things (you'll be able to do it later if needed):
* The `dasklimits` part.
* The `c.Backend.cluster_options` limit of dask-gateway workers.
* The Docker image used for Jupyter notebooks and Dask, please search for `pangeo/pangeo-notebook`.
  You might want to change either the image or just the associated tag.
* The Jupyter notebook resources limit in `singleuser`.

```yaml
dask-gateway:
  enabled: true
  gateway:
    auth:
      jupyterhub:
        apiToken: token1 # replace this 
      type: jupyterhub
    extraConfig:
      dasklimits: |
        c.ClusterConfig.cluster_max_cores = 6
        c.ClusterConfig.cluster_max_memory = "24 G"
        c.ClusterConfig.cluster_max_workers = 4
        c.ClusterConfig.idle_timeout = 1800
      optionHandler: |
        from dask_gateway_server.options import Options, Integer, Float, String

        def options_handler(options):
          if ":" not in options.image:
            raise ValueError("When specifying an image you must also provide a tag")
          return {
            "worker_cores": options.worker_cores,
            "worker_memory": int(options.worker_memory * 2 ** 30),
            "image": options.image,
          }

        c.Backend.cluster_options = Options(
          Integer("worker_cores", default=1, min=1, max=4, label="Worker Cores"),
          Float("worker_memory", default=2, min=2, max=8, label="Worker Memory (GiB)"),
          String("image", default="pangeo/pangeo-notebook:2022.09.21", label="Image"),
          handler=options_handler,
        )
dask-kubernetes:
  enabled: false
jupyterhub:
  hub:
    config:
      GenericOAuthenticator:
        client_id: egi_oauth_client # replace this 
        client_secret: egi_oauth_secret # replace this 
        oauth_callback_url: https://pangeo.vm.fedcloud.eu/hub/oauth_callback # replace this with your DNS name
        authorize_url: https://aai-dev.egi.eu/auth/realms/egi/protocol/openid-connect/auth
        token_url: https://aai-dev.egi.eu/auth/realms/egi/protocol/openid-connect/token
        userdata_url: https://aai-dev.egi.eu/auth/realms/egi/protocol/openid-connect/userinfo
        login_service: EGI Check-In
        scope:
          - openid
          - email
          - profile
          - eduperson_entitlement
        username_key: preferred_username
        userdata_params:
          state: state
        allowed_groups:
          - urn:mace:egi.eu:group:vo.pangeo.eu:role=member#aai.egi.eu
        claim_groups_key: eduperson_entitlement
      JupyterHub:
        authenticator_class: generic-oauth
    services:
      dask-gateway:
        apiToken: token1 # replace this
  ingress:
    annotations:
      kubernetes.io/ingress.class: nginx
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
    enabled: true
    hosts:
      - pangeo.vm.fedcloud.eu # replace this with your DNS name
    tls:
      - hosts:
        - pangeo.vm.fedcloud.eu # replace this with your DNS name
        secretName: pangeo.vm.fedcloud.eu # replace this with your DNS name
  proxy:
    secretToken: token2 # replace this 
    service:
      type: ClusterIP
  singleuser:
    cpu:
      guarantee: 1
      limit: 2
    defaultUrl: /lab
    extraEnv:
      DASK_GATEWAY__CLUSTER__OPTIONS__IMAGE: '{JUPYTER_IMAGE_SPEC}'
    image:
      name: pangeo/pangeo-notebook
      tag: 2022.09.21
    memory:
      guarantee: 2G
      limit: 4G
    startTimeout: 600
    storage:
      capacity: 2Gi
      type: dynamic
rbac:
  enabled: true
```

Here is the `helm` command to apply the changes (you might want to update or change helm chart version):

```bash
sudo helm upgrade daskhub daskhub \
    --repo=https://helm.dask.org \
    --install --wait \
    --cleanup-on-fail \
    --create-namespace \
    --namespace daskhub \
    --version 2022.8.2 \
    --values values.yaml
```

If all went well, JupyterHub will be available at
[https://pangeo.vm.fedcloud.eu/](https://pangeo.vm.fedcloud.eu/)


All members of the `vo.pangeo.eu` VO will be able to log into
JupyterHub with Check-In now at the DNS name created in [Step 1](#step-1-dns-name)
(e.g. [https://pangeo.vm.fedcloud.eu/](https://pangeo.vm.fedcloud.eu/)).

## Appendix

Here we collect additional information that may be helpful to reconfigure
the Pangeo cluster.

### Troubleshooting the Elastic Kubernetes functionnality

If things are not working as expected, you might want to look at 
the CLUES logs (`/var/log/clues2/clues2.log`).

#### OIDC Token expired

If you see some `OIDC auth Token expired` message in the file (which might 
happen right after the Kubernetes deployment), you'll need to renew manually
the OIDC token. To do so:

1. Login to https://im.egi.eu using EGI Checkin.
1. Go to menu Advanced -> Settings and copy the OIDC Access token value.
1. SSH to the cluster,
1. Remove `/usr/local/ec3/refresh.dat`.
1. Edit `/usr/local/ec3/auth.dat file`, copy your OIDC token in two places.
1. Wait a few minutes, new file refresh.dat must appear, and the 
   `OIDC auth Token expired` message from the log should disappear.

#### Cluster still not scaling

You need to check in /etc/clues2/conf.d/plugin-kubernetes.cfg that variables are correct,
especially variables related to current flavor of WN VMs.

For example, for a flavor with 8 cores and 32GiB:

```
KUBERNETES_NODE_MEMORY=33567285248
KUBERNETES_NODE_SLOTS=8
```

Then, you'll need to restart clues2 service:

```bash
service cluesd restart
```

#### Why cluster is not scaling down?

CLUES will only delete VMs that are _"idle"_: i.e. with __no running pods__.

You can use `kubectl` commands to see which pods are running on nodes:

```bash
sudo kubectl get nodes # get the list of nodes
kubectl describe nodes vnode-4.localdomain # shows the list of pods running on vnode-4
```

Keep also in mind that if you have configured a minimum number of free nodes, 
the same number of idle nodes will be maintained.

### Daskhub without EGI Check-in auth and less limits

If you don't want to go through the step of configuring EGI Check-in auth
for developement purpose, you can chose another authentication method,
for example NativeAuthenticator.

You might also want to suppress all the limitation on Dask cluster and workers,
have more resources on you Jupyterlab notebook server,
and configure the latest Pangeo images for Jupyter and dask.

Please use the `values.yaml` below to so:

```yaml
dask-gateway:
  enabled: true
  gateway:
    auth:
      jupyterhub:
        apiToken: token1 # replace this 
      type: jupyterhub
    extraConfig:
      optionHandler: |
        from dask_gateway_server.options import Options, Integer, Float, String

        def options_handler(options):
          if ":" not in options.image:
            raise ValueError("When specifying an image you must also provide a tag")
          return {
            "worker_cores": options.worker_cores,
            "worker_memory": int(options.worker_memory * 2 ** 30),
            "image": options.image,
          }

        c.Backend.cluster_options = Options(
          Integer("worker_cores", default=1, min=1, max=8, label="Worker Cores"),
          Float("worker_memory", default=4, min=2, max=32, label="Worker Memory (GiB)"),
          String("image", default="pangeo/pangeo-notebook:latest", label="Image"),
          handler=options_handler,
        )
dask-kubernetes:
  enabled: false
jupyterhub:
  hub:
    config:
      Authenticator:
        admin_users:
        - admin
      JupyterHub:
        admin_access: true
        authenticator_class: nativeauthenticator.NativeAuthenticator
    extraConfig:
      10-auth-config: |
        import os, nativeauthenticator
        c.JupyterHub.template_paths = [f"{os.path.dirname(nativeauthenticator.__file__)}/templates/"]
    services:
      dask-gateway:
        apiToken: token1 # replace this
  ingress:
    annotations:
      kubernetes.io/ingress.class: nginx
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
    enabled: true
    hosts:
      - pangeo.vm.fedcloud.eu # replace this with your DNS name
    tls:
      - hosts:
        - pangeo.vm.fedcloud.eu # replace this with your DNS name
        secretName: pangeo.vm.fedcloud.eu # replace this with your DNS name
  proxy:
    secretToken: token2 # replace this 
    service:
      type: ClusterIP
  singleuser:
    cpu:
      guarantee: 2
      limit: 4
    defaultUrl: /lab
    extraEnv:
      DASK_GATEWAY__CLUSTER__OPTIONS__IMAGE: '{JUPYTER_IMAGE_SPEC}'
    image:
      name: pangeo/pangeo-notebook
      tag: latest
    memory:
      guarantee: 4G
      limit: 8G
    startTimeout: 600
    storage:
      capacity: 2Gi
      type: dynamic
rbac:
  enabled: true
```

You'll need the same `helm` command as above to apply the changes:

```bash
sudo helm upgrade daskhub daskhub \
    --repo=https://helm.dask.org \
    --install --wait \
    --cleanup-on-fail \
    --create-namespace \
    --namespace daskhub \
    --version 2022.8.2 \
    --values values.yaml
```


### Using Daskhub Tosca template with Kubernetes

You can use Daskhub Tosca template to deploy a Daskhub platform. 
However, it won't be configured with EGI Checkin, and it may lack some settings as
it has not been updated since July 2022. If using it, you'lll probably have to
use Helm commands anyway.

You can use it by selecting it as Kubernetes option, and filling the options:

* `Dask Data` tab:
  * `Jupyterhub auth token`: please configure an auth token (e.g.
    with `openssl rand -hex 32` on Linux)
  * Use `Jupyterhub singleuser image` and `Jupyterhub singleuser image version`
    to configure the default user environment in JupyterHub with a container
    image of your choice.
