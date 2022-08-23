# How to deploy Pangeo in the infrastructure of the EGI Federation

These are the steps to deploy [Dask Gateway](https://gateway.dask.org/) using the 
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

1. Configure a DNS name for Pangeo using the
[Dynamic DNS](https://docs.egi.eu/users/compute/cloud-compute/dynamic-dns/) service.
1. Deploy a Kubernetes cluster on top of OpenStack, and install
the [DaskHub helm chart](https://helm.dask.org/).
[Infrastructure Manager Dashboard](https://docs.egi.eu/users/compute/orchestration/im/dashboard/)
(or simply IM Dashboard) will do this all for us automatically.
1. Configure JupyterHub with [EGI Check-In](https://docs.egi.eu/users/aai/check-in/) to allow
access to all members of the `vo.pangeo.eu` Virtual Organization.

### Step 1) DNS name

Log into the [Dynamic DNS web GUI portal](https://nsupdate.fedcloud.eu/)
with your EGI Check-In account to configure a domain name of your choice.
The web portal is intuitive, and there is also the associated
[documentation](https://docs.egi.eu/users/compute/cloud-compute/dynamic-dns/)
so we will not go into more details here.

For the rest of this tutorial, let us consider the `pangeo.vm.fedcloud.eu`
domain name.

### Step 2) Deploy Pangeo

Log into the [IM Dashboard](https://im.egi.eu/im-dashboard/)
with your EGI Check-In account, and configure your
[credentials](https://docs.egi.eu/users/compute/orchestration/im/dashboard/#cloud-credentials)
with the `vo.pangeo.eu` VO. Then click on the
[kubernetes template](https://im.egi.eu/im-dashboard/configure?selected_tosca=kubernetes.yaml)
and add `Dask`. Out of all the configuration options, please pay special
attention to the following:

* `Kubernetes Data` tab:
  * `Flag to install Cert-Manager` must be set to `True`
  * `Email to be used in the Let's Encrypt issuer`: add your preferred email.
  * `DNS name of the public interface of the FE node to generate the certificate`
       must be set to the one configure in [Step 1](#step-1-dns-name).
* `Dask Data` tab:
  * `Jupyterhub auth token`: please configure an auth token (e.g.
    with `openssl rand -hex 32` on Linux)
  * Use `Jupyterhub singleuser image` and `Jupyterhub singleuser image version`
    to configure the default user environment in JupyterHub with a container
    image of your choice.
* `Cloud Provider Selection` tab:
  * Select the [credentials](https://docs.egi.eu/users/compute/orchestration/im/dashboard/#cloud-credentials)
    configured earlier.

Please review all configuration options and click `Submit` and wait for the
deployment to finish with status `configured`. Then click on `Outputs` and
take a note of the IP address assigned. See the
[documentation](https://docs.egi.eu/users/compute/orchestration/im/dashboard/#infrastructures)
for IM Dashboard to learn more.

#### HTTPS

Please go to the [Dynamic DNS web GUI portal](https://nsupdate.fedcloud.eu/)
and update the public IP for the DNS name with the one shown in the `Outputs`
button of the IM Dashboard. Now `Reconfigure` the deployment so `https` is
correctly configured with Let's Encrypt.

You also need to update the nginx ingress for `https` to work. Here is
a template yaml file that should help:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: daskhub
  name: jupyterhub
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    kubernetes.io/ingress.class: nginx
spec:
  tls:
  - hosts:
    - pangeo.vm.fedcloud.eu
    secretName: pangeo.vm.fedcloud.eu
  rules:
  - host: pangeo.vm.fedcloud.eu
    http:
      paths:
      - backend:
          service:
            name: proxy-public
            port:
              name: http
        path: /
        pathType: Prefix
status:
  loadBalancer:
    ingress:
    - ip: <internal-ip-master>
```

Get the internal IP address of the master node with:

```bash
sudo kubectl get nodes -o wide
```

And update the ingress with:

```bash
sudo kubectl apply -f ingress.yaml -n daskhub
```

If all went well, JupyterHub will be available at
[https://pangeo.vm.fedcloud.eu/](https://pangeo.vm.fedcloud.eu/)

### Step 3) Configure EGI Check-In

Please follow the steps on the Check-In documentation for
[Service Providers](https://docs.egi.eu/providers/check-in/sp/). In particular,
the instructions using OpenID Connect as the authentication and authorization
protocol.

Check-In runs three separate instances: production (`aai.egi.eu`),
demo (`aai-demo.egi.eu`), and development (`aai-dev.egi.eu`). To quickly test
integration with Check-In, we suggest to configure Pangeo to connect to
the development instance. Note that by default the `vo.pangeo.eu` VO only exists
in the production instance of Check-In. Please ask the EGI Check-In team (via
an email to `check-in@egi.eu`) to create the VO in the development instance
of Check-In.

Here are additional details to fill out the registration form in the
[EGI Federation Registry](https://aai.egi.eu/federation/):

* `General` tab:
  * `Integration environment`: `Development`
* `Protocol Specific` tab:
  * `Select Protocol`: `OIDC Service`
  * `Client ID`: leave this empty
  * `Application Type`: `Web`
  * `Redirect URI`: `https://pangeo.vm.fedcloud.eu/hub/oauth_callback`
  * `Scope`: select `openid`, `email`, `profile`, `eduperson_entitlement`,
     `eduperson_scoped_affiliation`, and `eduperson_unique_id`.
  * `Grant Types`: `authorization code`
  * `Token Endpoint Authorization Method`: `Client Secret over HTTP Basic`
  * `Client Secret`: leave this empty

After you have successfully registered the Pangeo service in the
[EGI Federation Registry](https://aai.egi.eu/federation), here is the
`values.yaml` that you need to connect Dask with Check-In:

```yaml
dask-gateway:
  enabled: true
  gateway:
    auth:
      jupyterhub:
        apiToken: <token> # Jupyterhub auth token generated above with openssl
      type: jupyterhub
    prefix: /services/dask-gateway
    extraConfig:
      dasklimits: |
        c.ClusterConfig.cluster_max_cores = 6
        c.ClusterConfig.cluster_max_memory = "24 G"
        c.ClusterConfig.cluster_max_workers = 3
        c.ClusterConfig.idle_timeout = 1800
    backend:
      worker:
        cores:
          limit: 2
        memory:
          limit: 8G
        threads: 2
  traefik:
    service:
      type: ClusterIP
dask-kubernetes:
  enabled: false
jupyterhub:
  hub:
    config:
      GenericOAuthenticator:
        client_id: <id>
        client_secret: <secret>
        oauth_callback_url: https://pangeo.vm.fedcloud.eu/hub/oauth_callback
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
    nodeSelector:
      node-role.kubernetes.io/master: ""
    services:
      dask-gateway:
        apiToken: <token> # Jupyterhub auth token generated above with openssl
    tolerations:
    - key: node-role.kubernetes.io/master
      operator: Exists
  proxy:
    chp:
      nodeSelector:
        node-role.kubernetes.io/master: ""
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
    service:
      type: ClusterIP
  singleuser:
    cpu:
      guarantee: 1
      limit: 2
    defaultUrl: /lab
    image:
      name: pangeo/ml-notebook
      tag: latest
    lifecycleHooks:
      postStart:
        exec:
          command:
          - sh
          - -c
          - |
            chmod 700 .ssh; chmod g-s .ssh; chmod 600 .ssh/*; exit 0
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

Here is the `kubectl` command to apply the changes:

```bash
sudo helm upgrade daskhub daskhub \
    --repo=https://helm.dask.org \
    --install \
    --cleanup-on-fail \
    --create-namespace \
    --namespace daskhub \
    --version 2022.6.0 \
    --values values.yaml
```

It looks like you need to reconfigure the ingress after applying the changes
above. Please re-run:

```bash
sudo kubectl apply -f ingress.yaml -n daskhub
```

All going well, all members of the `vo.pangeo.eu` VO will be able to log into
JupyterHub with Check-In now at the DNS name created in [Step 1](#step-1-dns-name)
(e.g. [https://pangeo.vm.fedcloud.eu/](https://pangeo.vm.fedcloud.eu/)).

## Appendix

Here we collect additional information that may be helpful to reconfigure
the Pangeo cluster.

### Revert Authentication Configuration

If you want to go back to the original configuration for user authentication
please use the `values.yaml` below:

```yaml
dask-gateway:
  enabled: true
  gateway:
    auth:
      jupyterhub:
        apiToken: <token> # Jupyterhub auth token generated above with openssl
      type: jupyterhub
    prefix: /services/dask-gateway
    extraConfig:
      dasklimits: |
        c.ClusterConfig.cluster_max_cores = 6
        c.ClusterConfig.cluster_max_memory = "24 G"
        c.ClusterConfig.cluster_max_workers = 3
        c.ClusterConfig.idle_timeout = 1800
    backend:
      worker:
        cores:
          limit: 2
        memory:
          limit: 8G
        threads: 2
  traefik:
    service:
      type: ClusterIP
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
    nodeSelector:
      node-role.kubernetes.io/master: ""
    services:
      dask-gateway:
        apiToken: <token> # Jupyterhub auth token generated above with openssl
    tolerations:
    - key: node-role.kubernetes.io/master
      operator: Exists
  proxy:
    chp:
      nodeSelector:
        node-role.kubernetes.io/master: ""
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
    service:
      type: ClusterIP
  singleuser:
    cpu:
      guarantee: 1
      limit: 2
    defaultUrl: /lab
    image:
      name: pangeo/ml-notebook
      tag: latest
    lifecycleHooks:
      postStart:
        exec:
          command:
          - sh
          - -c
          - |
            chmod 700 .ssh; chmod g-s .ssh; chmod 600 .ssh/*; exit 0
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

Here is the `kubectl` command to apply the changes:

```bash
sudo helm upgrade daskhub daskhub \
    --repo=https://helm.dask.org \
    --install \
    --cleanup-on-fail \
    --create-namespace \
    --namespace daskhub \
    --version 2022.6.0 \
    --values values.yaml
```

Remember to reconfigure the ingress after applying the changes
above if you run into issues:

```bash
sudo kubectl apply -f ingress.yaml -n daskhub
```
