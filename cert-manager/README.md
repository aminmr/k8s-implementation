# Cert-Manager

In this section, I wrote about how to implement the `cert-manager` for issuing Services certificates.

If you installed the Kubernetes via `kubespray` you can use the `addons.yaml` and change the cert-manager installation to true. Otherwise, you can install the cert-manager manually:( At the time of writing this document the latest version of cert-manager was v1.6.1)

```shell
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
```

After installing the cert-manager, to issue any certificates, you’ll need to configure an [`Issuer`](https://cert-manager.io/docs/configuration/) or [`ClusterIssuer`](https://cert-manager.io/docs/configuration/) resource first.

**ClusterIssuer: **ClusterIssuer is **used to issue certificates in any** namespace

**Issuer: **Issuer is a namespaced resource allowing you to use different CAs in each namespace

So we need to deploy the issuer. For doing this you can apply `issuer.yaml`. If you want to create the `ClusterIssuer` you just need to change the manifest type.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: amin79mr@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: tls-secret
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```

 	 ## Usage

You can request to issue the certificate in ways.

### 1. Ingress-resource

 The simple way is to use `annotations` in ingress-resource you were written:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
     kubernetes.io/ingress.class: "nginx"
     cert-manager.io/cluster-issuer: "letsencrypt"
spec:
  tls: 
    - hosts:
        - nginx.aminmr.ir
      secretName: nginx-ssl
  rules:
  - host: nginx.aminmr.ir
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-deployment
            port:
              number: 80
```

So in the above manifest, I request for Let's Encrypt SSL via annotations. After applying this ingress-resource, the request has been sent to the cert-manager and the SSL issuing process will be started.

For further read click on the following link:

[TLS Automated Certficate Management](https://kubernetes.github.io/ingress-nginx/user-guide/tls/#automated-certificate-management-with-cert-manager)

### 2. Certificate Resource

If you want to issue a certificate first you need to apply a `Certificate` resource:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: acme-crt
spec:
  secretName: acme-crt-secret
  dnsNames:
  - example.com
  - foo.example.com
  issuerRef:
    name: letsencrypt-prod
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: Issuer
    group: cert-manager.io
```

And after applying the above manifest the issuing process will be started.

For further read click on the following link:

[Certificate Resource](https://cert-manager.io/docs/concepts/certificate/)

## Troubleshooting

If you have any problem in issuing the certificate, the cert-manager has a guide for troubleshooting. You can read it step by step and find your problem.

In my case after all steps, the http-solver pod was created but it remains for minutes. I follow the troubleshooting page and by reading the logs I understood that the OS DNS resolution got a problem. 

To access the page you can click on the following link:

[Cert-Manager Troubleshooting](https://cert-manager.io/docs/faq/acme/)

