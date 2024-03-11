# Gitlab Docs

## Gitlab Private Server Setup

This is document for setting Gitlab Private Server. 

### Prerequisite
* Kubernetes Cluster
* Filesystem Storage Class

### System Requirements
* minimum 4 core CPU
* minimum 10 GB ram
* minimum 5 GB storage

### Installation Procedure

#### Step One: Deploy resources (PVC, Service, Ingress)
First we need to deploy the gitlab descriptors below.

````
kubectl apply -f gitlab-pvc.yaml
kubectl apply -f gitlab-svc.yaml
kubectl apply -f gitlab-ingress.yaml
````

1. [gitlab-pvc.yaml](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/descriptors/gitlab/gitlab-pvc.yaml)

> Note: Here in pvc.yaml, the storage class will be filesystem because deployment will try to attach storage.

2. [gitlab-svc.yaml](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/descriptors/gitlab/gitlab-svc.yaml)
3. [gitlab-ingress.yaml (extensions/v1beta1)](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/descriptors/gitlab/gitlab-ingress(extensions-v1beta1).yaml) or
   [gitlab-ingress.yaml (networking.k8s.io/v1)](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/descriptors/gitlab/gitlab-ingress(networking.k8s.io-v1).yaml)

> Note: Here in ingress.yaml, there is not tls enabled because we will configure tls certificates
> from inide the gitlab server. Just set the domain and we will setup the wild-certificate secrets following steps. 

#### Step Two: Install Deployment

Deploy the following deployment file

````
kubectl apply -f gitlab-deploy.yaml
````

1. [gitlab-deploy.yaml](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/descriptors/gitlab/gitlab-deploy.yaml)

There are certain components inside deployments:

* `spec.template.spec.serviceAccountName` (Optional) if your cluster is psp (Pod Security Policy) enabled, 
  which means the descriptor needs permission to access the initiate then it requires. Here the value 
  is `ns-privileged-sa`. To setup service account and bind roles to it deploy the following
  descriptors (if its not setup yet).
  
  * [service-account.yaml](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/descriptors/rbac/service-account.yaml)
  * [clusterrole.yaml](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/descriptors/rbac/clusterrole.yaml)
  * [crb.yaml](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/descriptors/rbac/crb.yaml)

* `spec.template.spec.volumes[].name: cert-files` (Required) this value is required to set the existing certificates
to gitlab. here the value `wild-certificate-secret` is the wild car secret name which contains certicate and private key for a domain.
  for this example lets say the wild car domain is `*.console.klovercloud.com` so that we can use `gitlab.console.klovercloud.com` as a subdomain
    

* `spec.template.spec.containers[].volumeMounts[].name: cert-files: cert-files` here we need to set the certificate and the private key to 
  gitlab into path under `/etc/gitlab/ssl` for example`/etc/gitlab/ssl/gitlab.example.com.crt` and `/etc/gitlab/ssl/gitlab.example.com.key`. here  you
  can set any value in place of `/gitlab.example.com`, we can set the domain `gitlab.console.klovercloud.com` instead or any value. 
  If the folder does not exists kubernetes will create it. **Remember that we need to set these to path to gitlab config file**

After deploying the deployment wait for pod to be running.

![](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/static/1.gitlab.PNG)

#### Step Three: Update the Gitlab Config and Restart Gitlab Server

Now we need to update the certificates and configurations to gitlab server.

First we need to `exec` into the gitlab pod.

```
kubectl exec -it <pod name> -n <namespace> bash 
```
![](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/static/2.gitlab.PNG)


Now after entering into the pod, open the config file ` /etc/gitlab/gitlab.rb`

```
 vi /etc/gitlab/gitlab.rb
```

![](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/static/3.gitlab.PNG)

Now we have to edit the following config to the file.

1. We need to set external url to our url. we need to set `https//<url>` since we want to secure the
connection
```
external_url 'https://gitlab.console.klovercloud.com'
```
![](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/static/4.gitlab-config.PNG)

2. We need to set the certificate and private key path and enable nginx https. **The path must be match with whatever we have set in the deployment 
descriptor**.

````
nginx['enable'] = true
nginx['client_max_body_size'] = '100m'
nginx['redirect_http_to_https'] = true
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.example.com.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.example.com.key"
nginx['ssl_protocols'] = "TLSv1.2 TLSv1.3"
````


![](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/static/5.gitlab-config.PNG)

> Note: In vim Editor to search a text write `/<searched text>` and press enter

![](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/static/6.shortcut-vim.PNG)

3. Now save and exit the file from vim editor

```
<esc button>
:x
```

4. Restart the server by the following command and press enter.
```
 gitlab-ctl reconfigure
```

![](https://github.com/shaekhhasanshoron/SSO-Keycloak/blob/main/static/7.gitlab-config.PNG)

5. Exit from the pod with the following command

```
exit
```

#### Step Four: Validate Url

Now check if `https://github.console.klovercloud.com` access the gitlab or not.
