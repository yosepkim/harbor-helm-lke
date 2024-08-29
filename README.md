# Installing Harbor on Linode Kubernetes Engine

This guide will take you through how to install Harbor using a Helm chart on Linode Kubernetes Engine (LKE). For this installtion, a helm chart curated by Bitnami will be used. 

### Why Bitnami?
Bitnami continously monitors and updates every application in their catalog including its components and dependencies to ensure their applications and development stacks are always current and secure. Also, their applications and stacks are pre-configured and ready-to-use immediately on any platform.

### Why LKE?
Linode Kubernetes Engine provides a fully-managed, easy-to-use, vanilla Kubernetes. It is also cost effective.

### and, Why Harbor?
Harbor is a well-maintaied, fully open-source, container registry project. Out of the box, Harbor provides static analysis of vulnerabilities in images through the open source projects [Trivy](https://github.com/aquasecurity/trivy) and [Clair](https://github.com/quay/clair). With the gentle but secure touch from Bitnami, Harbor has been a popular choice for enterprise across industries.


## Let's get our hands dirty

### Prerequisites
Make sure that the following tools are installed on your machine:
1. kubectl
2. docker (dekstop)

### Create a LKE cluster
Creating a K8s cluster on LKE is super simple. Once you log in, click on the `Kubernetes` button on the side menu, then click on the `Create Cluster` button. The rest is self explanatory. If you get stuck, refer to this [article](https://techdocs.akamai.com/cloud-computing/docs/create-a-cluster)

By now, we hope you have a working cluster!

You can download the `kubeconfig.yaml` file from the UI. Then you can run the following commands to check if you can access the cluster: 

```
    $ export KUBECONFIG=/path/to/{kubeconfig.yaml}
    $ kubectl get all
```


Before we deploy Harbor, let's do some planning. We first need to think about how users would access Harbor. Once you decide on a fully qualified domain name(FQDN), let's think about how we can secure it. That is, we need to create a TLS certificate.

### Create SSL certificate

Although we can use self-signed certificate or have Harbor generate certificates for us, to make it easy for users to be able to access Harbor it is best practice to use CA-signed certificates.

The following example commands show you how to generate a certificate on Mac:

```
    $ brew install python-cryptography
    $ brew install certbot
    $ certbot certonly --server https://acme-v02.api.letsencrypt.org/directory --manual --preferred-challenges dns -d '{your-domain}'
```

Now we are ready to create a `secret` resource that would contain the certificate and private key file that you just generated:

```
    $ kubectl create secret tls harbor-tls --cert=/path/to/certificate.pem --key=/path/to/key.pem
```

## Install Harbor using Helm

Before installing Harbor, let's make sure you review the `harbor-config.yaml` file that is in this repository. A rough explanation can be found in the [README](https://artifacthub.io/packages/helm/bitnami/harbor). 

To ensure that Harbor gets installed successfully on LKE, there is a couple of places we need to make adjustments to:

#### 1. `exposureType` needs to be `proxy` and `service.type` should be `loadBalancer`
This is to ensure that a nodeBalancer gets created property.

```
exposureType: proxy
## Service parameters
##
service:
  ## @param service.type NGINX proxy service type
  ##
  type: LoadBalancer
  ## @param service.ports.http NGINX proxy service HTTP port
```

#### 2. `nginx.tls.existingSecret` must be set to the secret you created earlier 

```
nginx:
  ...
  tls:
    enabled: true
    existingSecret: "harbor-tls"
```

With these changes, you can run the following command to deploy Harbor to LKE:

```
    $ helm install harbor-repo bitnami/harbor --version 23.0.1 -f harbor-config.yaml
```

After waiting for a few seconds, you now can get the external IP address by which you can access the UI portal:

```
    $ kubectl get svc --namespace default -w harbor-repo
```

### Update your DNS record
You can now add an A record to point your FQDN to the Harbor via the IP address. If you manage your domains in Linode, this [guide](https://techdocs.akamai.com/cloud-computing/docs/manage-dns-records) will walk you through the process. 


### Access the protal
You can use the following command to get the initial password 

```
    $ kubectl get secret --namespace default harbor-repo-core-envvars -o jsonpath="{.data.HARBOR_ADMIN_PASSWORD}" | base64 -d
```

Using `admin` as a user and the password retrieved from the previous step, you may now open the browser and type in `https://{your-FQDN}` to access the Harbor portal.

Once logged in, you can create a project to which you can push your container images.

### Push your first container image 

Just like how you would normally push images to Docker Hub, you first need to log in to the Harbor repository.

```
    $ docker login https://{your-FQDN}
```

Run the followign command to create a tag then push it to the Harbor.

```
    $ docker tag {your-image} core.harbor.domain/umpqua/{your-image}:latest
Push the image to Harbor
    $ docker push core.harbor.domain/umpqua/hello-world-java:latest
```

Yay! Congratulations. You now have a private container image repository!

You can now specify a container image from Harbor in your Kubernetest deployment / pod definition files to have it pull it from Harbor. 
