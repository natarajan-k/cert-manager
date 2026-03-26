# cert-manager

cert-manager is an open-source tool designed to automate the management and issuance of TLS certificates within Kubernetes environments. It integrates with certificate authorities like Let's Encrypt to automatically request, renew, and store certificates, reducing the need for manual intervention. By handling certificate lifecycle tasks seamlessly, cert-manager helps ensure secure communication between services and simplifies HTTPS enablement for applications running in clusters.

## References

1. [https://cert-manager.io/docs/](https://cert-manager.io/docs/).  
2. [cert-manager Operator for Red Hat OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift).  
3. [Let's Encrypt](https://letsencrypt.org/docs/). 

## Purpose of this Document.  

In this document I am going to highlight the steps to install cert-manager and create certificates. I'll have the steps for creating self-signed certificates as well integration with Let's Encrypt.  

## Installing cert-manager

Simplest way is to install cert-manager with the default static configurations.  

    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/<VERSION>/cert-manager.yaml
By default, cert-manager will be installed in the cert-manager namespace. You can verify installation by checking the pods in the cert-manager namespace. 

    kubectl -n cert-manager get pods.  

You should see the cert-manager, cert-manager-cainjector and cert-manager-webhook pods in 'Running' state. 


## Create ClusterIssuer

First thing to setup after installing cert-manager is an Issuer or a ClusterIssuer.    
An Issuer is a namespaced resource. You can only create certificates for that particular namespace.    
A ClusterIssuer allows you to create certificates for multiple namespaces.    

### Self-Signed ClusterIssuer

A self-signed ClusterIssuer in cert-manager is used to generate certificates that are signed by the cluster itself, without relying on an external Certificate Authority. It is typically used for internal communication, testing, etc.   
It is not recommended for external or production systems communication.  
This is a sample yaml for creating a self-signed clusterissuer. 

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

### Let's Encrypt ClusterIssuer.  

Using Let's Encrypt with a ClusterIssuer in cert-manager enables automatic provisioning and renewal of trusted TLS certificates from a public Certificate Authority for your applications.  
In Let's Encrypt, the challenge process verifies domain ownership by requiring you to complete a specific task-such as serving a token over HTTP (HTTP-01) or adding a DNS record (DNS-01) -to prove control of the domain. Hence, it is mandatory that your server is accessible via it's public facing DNS name.   
This is a sample yaml for creating a let's encrypt clusterissuer.   

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: <YOUR EMAIL>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: <INGRESS CLASS>
```  

To obtain the ingress class: 

    kubectl get ingressclass


## Create Certificates

Next step is to create certificates. In cert-manager, a Certificate resource defines the desired TLS certificate, including the domain names, secret name, and the issuer to use.
cert-manager continuously reconciles this resource by requesting, renewing, and storing the certificate in the specified Kubernetes Secret.  


```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kafka-letsencrypt
spec:
  secretName: kafka-letsencrypt
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
    group: cert-manager.io
  subject:
    organizations:
      - apps.itz-36mg6h.infra01-lb.tok04.techzone.ibm.com
  dnsNames:
    - bootstrap.apps.itz-36mg6h.infra01-lb.tok04.techzone.ibm.com    
    - kafka1.apps.itz-36mg6h.infra01-lb.tok04.techzone.ibm.com  
    - kafka2.apps.itz-36mg6h.infra01-lb.tok04.techzone.ibm.com   
    - kafka3.apps.itz-36mg6h.infra01-lb.tok04.techzone.ibm.com   

```

Important fields to take note of:   
Name of the clusterissuer to use (e.g. let's encrypt or self-signed).    
The secret where the generated certificates should be stored.    
List of DNS names. This is the list of DNS that should be included in the certificate (list of SAN).  
You can use wildcard DNS if using the self-signed clusterissuer. 
With Let's Encrypt clusterissuer, wildcard entries are only allowed if you use the DNS-01 challenge. 

Check status of certificates:

    kubectl  get certificates -o wide

You should see a Status.Condition "Certificate is up to date and has not expired" for a certificate that has been issued and is ready to be used.   

You can now use the secret where the certificate is stored as a secretRef in your applications.  