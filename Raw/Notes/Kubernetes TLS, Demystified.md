---
title: Kubernetes TLS, Demystified
source: https://dev.to/otomato_io/possible-paths-2hfc
clipped: 2023-09-04
published: 2022-10-11
category: network
tags:
  - k8s
  - network
  - tls
read: false
---

> This is the anniversary 10th article in this series.

🛡️ It is more than obvious a secured connection to any exposed service running in Kubernetes cluster is important.

The supposition for this article is that you wish to set up TLS (Transport Layer Security) realm for your [ingress resource](https://docs.nginx.com/nginx-ingress-controller/) and that you already have a functioning ingress controller established in your cluster.

> The SSL replacement technology is called Transport Layer Security (TLS) today. An enhanced version of SSL is TLS.

Similar to how SSL operates, it uses encryption to safeguard the transmission of data and information. Although SSL is still commonly utilized in the industry, the *two names are frequently used interchangeably*.

## [](#getting-a-certificate-what-paths-can-be-taken)Getting a certificate: what paths can be taken?

A TLS/SSL certificate is the fundamental prerequisite for ingress TLS. These certificates are available to you in the following ways.

-   **Path one**. Self-signed certificates: Our own Certificate Authority (root CA) [created and signed](https://www.ibm.com/docs/en/api-connect/10.0.1.x?topic=overview-generating-self-signed-certificate-using-openssl) the TLS certificate. It is a well-known, stunt choice for *testing scenarios* where you can "collaborate" on the root CA so that browsers will accept the certificate.

[![[Raw/Media/Resources/a2d2a0c109960091b4bec4017d17250a_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--eXGN7HTC--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x8ol9xl8v4idbbuh8ahf.png)  
🖼️ *Pic source: Bizagi*

-   **Path two**. Get an SSL certificate: for production use cases, you must [purchase](https://www.google.com/search?q=buy+ssl+certificate) an SSL certificate from a reputable certificate authority that operating systems and browsers trust. But you must bear in mind that a so-called *wildcard certificate* suitable to protect all subdomains in a domain can cost $300+/year from major commercial issuers.
    
-   **Path three**. Use a Let's Encrypt certificate: Let's Encrypt is a reputable certificate authority that issues *free* TLS certificates.
    

### [](#a-few-words-about-lets-encrypt)A few words about Let's Encrypt

It is a non-profit organization founded by [enthusiasts](https://letsencrypt.org/2022/09/12/remembering-peter-eckersley.html) in the field of struggle for privacy and security in 2014.

The challenge–response protocol used to automate enrolling with the certificate authority is called Automated Certificate Management Environment ([ACME](https://letsencrypt.org/how-it-works/)). It can query either Web servers or DNS servers controlled by the domain covered by the certificate to be issued.

[![[Raw/Media/Resources/825553e25ebf60c1b39af0c48573af8e_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--WY4coUei--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uzzu0229x0x0tye8d3fk.png)  
🖼️ *Pic source: Let's Encrypt*

> If you are interested in the implementation of the protocol, read what [Adrian Colyer](https://blog.acolyer.org/2020/02/12/lets-encrypt-an-automated-certificate-authority-to-encrypt-the-entire-web/) from SpringSource writes about them.

Each SSL certificate has an *expiration date*. So, before the certificate expires, you need to *rotate* it. For instance, Let's Ecrypt certificates have a *three-month* expiration date (and [here](https://letsencrypt.org/2015/11/09/why-90-days.html) they tell why).

This way, below in this article series, the author will dwell on the **third path** further in detail. Why? Well, since this path is interesting for its relative self-sufficiency and \[relative\] independence from commercial / state-owned certificate issuers. In general, the motto of this path is: "If made something with your hands, you know how it works - you're more adapted to survival!"

Of course, Let's Encrypt approach does not *always* fit needs, but for academic purposes and for startups it works.

But let's look at the situation "in manual mode" - just try to associate a certificate with a protected application. So,

### [](#chicken-or-egg)Chicken or egg?

The *ingress controller*, not the ingress resource, is in charge of SSL. In other words, the ingress controller *accesses* the TLS certificates you provide to the ingress resource as a Kubernetes `secret` and incorporates them into its configuration.

[![[Raw/Media/Resources/2fa4766dbf6983491a20188a6d798c12_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--W6HGQZQr--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s68iypo5revwa9w665am.png)

## [](#setup-tlsssl-certificates-for-ingress)Setup TLS/SSL certificates for ingress

Let's examine the procedures for setting up TLS for ingress. We'll start by launching a test application on the cluster. This application will be used to test our TLS-secured ingress.

Establish the new namespace, `trial`.  

```
kubectl create namespace trial
```

Enter fullscreen mode Exit fullscreen mode

Keep this as `hello-app.yaml`. The `Deployment` and `Service` objects are present.  

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: trial
spec:
  selector:
    matchLabels:
      app: hello
  replicas: 1
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: "rbalashevich/hello-app:2.0"
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
  namespace: trial
  labels:
    app: hello
spec:
  type: ClusterIP
  selector:
    app: hello
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
```

Enter fullscreen mode Exit fullscreen mode

Deploy the application with a command:  

```
kubectl apply -f hello-app.yaml -n trial
```

Enter fullscreen mode Exit fullscreen mode

### [](#create-a-kubernetes-tls-secret)Create a Kubernetes TLS Secret

It is necessary to make the SSL certificate a Kubernetes secret. It will subsequently be directed to the `tls` block for `Ingress` resources.

The `server.crt` (CA trust chain) and `server.key` (private key) SSL files are assumed to be available from a Certificate Authority, your company, or self-signed, as a last resort.

⚠️ A private key is created by you (the certificate owner) when you request your certificate with a Certificate Signing Request (CSR). Saying other words, you receive a private key when generate a CSR. You submit the CSR code to the certificate authority and keep private key in a safe place.

As for the big three public cloud providers, they have instructions for exporting certificates: [AWS CM](https://docs.aws.amazon.com/acm/latest/userguide/export-private.html), [GCP CAS](https://cloud.google.com/sdk/gcloud/reference/privateca/certificates), [Azure KV](https://learn.microsoft.com/en-us/azure/key-vault/certificates/how-to-export-certificate?tabs=azure-cli).

It is necessary to make the SSL certificate a Kubernetes secret. It will subsequently be directed to the `tls` block for `Ingress` resources.

> And yes, keep the private key (`server.key`)!

Let's use the `server.crt` and `server.key` files to construct a Kubernetes secret of `tls` type (SSL certificates). In the `trial` namespace, where the `hello-app` deployment is located, we are creating the secret.

Run the `kubectl` command listed below from the directory where your server is located. Supply the *absolute path* to the files or the `.crt` and `.key` files. The name `hello-app-tls` is made up.  

```
kubectl create secret tls hello-app-tls \
    --namespace trial \
    --key server.key \
    --cert server.crt
```

Enter fullscreen mode Exit fullscreen mode

The comparable YAML file, where you must include the contents of the `.crt` and `.key` files, is provided below.  

```
apiVersion: v1
kind: Secret
metadata:
  name: hello-app-tls
  namespace: trial
type: kubernetes.io/tls
data:
  server.crt: |
       <crt contents here>
  server.key: |
       <private key contents here>
```

Enter fullscreen mode Exit fullscreen mode

A Kubernetes ingress is [a set of rules](https://kubernetes.io/docs/concepts/services-networking/ingress/) that can be configured to give services externally reachable URLs. Based on this understanding, to turn on secure connection, we should add `tls` block to `Ingress` object. So, in the `trial` namespace, we create the sample ingress TLS-capable resource:  

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-app-ingress
  namespace: trial
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.hosting.cloudprovider.com
    secretName: hello-app-tls
  rules:
  - host: "app.hosting.cloudprovider.com"
    http:
      paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: hello-service
              port:
                number: 80
```

Enter fullscreen mode Exit fullscreen mode

⚠️ Replace `app.hosting.cloudprovider.com` to your actual hostname. The `host(s)` should be the same in both the `rules` and `tls` blocks in the `Ingress` manifest. In other words, they must match.

In case of using NGINX ingress, you can add the supported annotation by the ingress controller you are using if you want a *strict* SSL. For instance, you can use the [annotation](https://github.com/kubernetes/ingress-nginx/blob/main/docs/user-guide/nginx-configuration/annotations.md) `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"` in the Nginx ingress controller to permit SSL traffic up until the application.

### [](#the-way-to-make-sure)The way to make sure

Let's check with `curl https://app.hosting.cloudprovider.com -kv`, is the connection to the app secure now:  

```
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=app.hosting.cloudprovider.com
*  start date: Oct 6 15:35:07 2022 GMT
*  expire date: Oct 6 15:35:07 2023 GMT
*  issuer: CN=Go Daddy Secure Certificate Authority - G2,
              OU=http://certs.godaddy.com/repository/,
              O="GoDaddy.com, Inc.",L=Scottsdale,ST=Arizona,C=US
*  SSL certificate verify ok.
```

Enter fullscreen mode Exit fullscreen mode

🔒 If the certificate is valid, then the browser will not swear and there will be no frightening warnings either. Voilà, the connection to our app is secure!

Okay, we've covered the situations for the first and second paths. The next step is pathfinding the third path involving Let's Encrypt certificate.

## [](#estne-vita-vere-brevis)Estne vita vere brevis?

Sed vita est cum dignitate vivendum. As the author noted above, the life of a certificate from a let's encrypt is [short](https://letsencrypt.org/2015/11/09/why-90-days.html). You have to pay for insolence. Accordingly, some solution is required that would automate the re-issuance of short-lived certificates, right? And such a solution exists, it is `cert-manager`! It streamlines the process of getting, renewing, and using certificates by adding certificates and certificate issuers as *resource types* in Kubernetes clusters.

It can generate certificates from a number of supported sources, including Let's Encrypt.

[![[Raw/Media/Resources/d4780b889d26a44e0e6f6f90c5992062_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--FfV4jEBR--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2w3rx680y8nvz18b9ms8.png)

Furthermore, it will check that certificates are current and valid and make an attempt to **renew** them for a specified period **before** they **expire**.

### [](#install-certmanager-on-kubernetes)Install cert-manager on Kubernetes

According to the official `cert-manager` documentation, you can install it by using [kubectl](https://cert-manager.io/docs/installation/kubectl/) or by the provided [helm](https://cert-manager.io/docs/installation/helm/) chart.  

```
# Create a dedicated Kubernetes namespace for cert-manager
kubectl create namespace cert-manager

# Add official cert-manager repository to helm CLI
helm repo add jetstack https://charts.jetstack.io

# Update Helm repository cache (think of apt update)
helm repo update

# Install cert-manager on Kubernetes
## cert-manager relies on several Custom Resource Definitions (CRDs)
helm install certmgr jetstack/cert-manager \
    --set installCRDs=true \
    --version v1.9.1 \
    --namespace cert-manager
```

Enter fullscreen mode Exit fullscreen mode

The `Issuer` is responsible for issuing certificates. It is the signing authority and based on its configuration. The issuer knows how certificate requests are handled.

Cert-manager also creates several objects using different specifications such as `CertificateRequest`.

A `Certificate` resource is a readable representation of a certificate request. Certificate resources are linked to an `Issuer` who is responsible for requesting and renewing the certificate.

To determine *if* a certificate *needs to be re-issued*, `cert-manager` looks at the the `spec` of `Certificate` resource and latest `CertificateRequests` as well as the data in `Secret` containing the certificate.

### [](#lets-encrypt-staging-or-production-server)Let's Encrypt: staging or production server?

An `Issuer` is a custom resource (CRD) which tells `cert-manager` how to sign a `Certificate`. Following [this howto (section 7)](https://cert-manager.io/docs/tutorials/getting-started-with-cert-manager-on-google-kubernetes-engine-using-lets-encrypt-for-ingress-ssl/) the `Issuer` will be configured to connect to the Let's Encrypt staging server, which allows us to test everything without using up your Let's Encrypt [certificate quota](https://letsencrypt.org/docs/rate-limits/) for the domain name.

After debugging, you can safely issue a certificate by using LE's production server.

A video describing `cert-manager` YAML syntax and recommended by the author of this article is [📽️ Anton Putra's good one](https://youtu.be/7m4_kZOObzw).

## [](#conclusion)Conclusion

SSL certificate acquisition was made simple by Let's Encrypt's reputation as a reliable certificate authority. Together with `cert-manager` tool, ops can quickly and easily assure correct transport encryption and interoperability with already-existing parts like NGINX Ingress. In addition to the example mentioned above, `cert-manager` can help with trickier situations like those involving wildcard SSL certificates.

If you're interested in using letsencrypt outside of a kubernetes cluster, take a look at [Caddy](https://github.com/caddyserver/caddy), a 43k ⭐ open source web server, and also at Certbot, a 29k ⭐ ACME client which is open source, too.

Ever tried using `wireshark` to monitor web traffic? Follow [Aaron Phillips](https://www.comparitech.com/net-admin/decrypt-ssl-with-wireshark/) from Comparitech to learn how.

Safe connections to you!