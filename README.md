# Cloudflare with DigitalOcean Kubernetes for DDoS, WAF, and Other Services

## Problem Statement
DigitalOcean offers a managed Kubernetes service (DOKS) with basic DDoS protection. When running an application in DOKS, users might require advanced DDoS protection, a Web Application Firewall (WAF), and other similar services that Cloudflare provides. This tutorial will describe how to integrate Cloudflare services with DOKS.

## Architecture
Let's imagine a web application running in DOKS. This application is a Kubernetes Deployment with a Service that publicly exposes it via DigitalOcean's managed load balancer. To access the application, a client connects via HTTPS to Cloudflare. Cloudflare inspects the request. If the request is malicious, Cloudflare drops it. If the request is legitimate, it gets proxied to DigitalOcean over HTTPS. The load balancer on DigitalOcean receives the request, terminates TLS, and redirects it unencrypted over HTTP to one of the worker nodes. Terminating TLS on a load balancer is convenient for maintaining the TLS certificate at a single central point, thus avoiding the need to manage TLS certificates in a decentralized manner on worker nodes. The application then responds to the client through the same chain of proxies.

## Configuration Prerequisites
Ensure the following tools are configured on your computer:
- doctl
- kubectl
- jq

## Configuration

### Adding a Site to Cloudflare
Have your domain name registered and ready. Use it to add a new site in Cloudflare. DNS records for this domain are going to be managed by Cloudflare. Ensure your authoritative DNS servers, or nameservers have been changed at your domain registrar (e.g., GoDaddy, Namecheap) to point to Cloudflare.

### Enabling HTTPs End-to-End
In Cloudflare control panel go to your domain > SSL/TLS > Edge Certificates and enable "Always Use HTTPS". This will ensure that the traffic will always get encrypted between the client and Cloudflare.

In Cloudflare control panel go to your domain > SSL/TLS > Overview and change encryption mode to "Full (strict)". This will ensure that the traffic will get encrypted not only between the client and Cloudflare, but also between Cloudflare and DigitalOcean.

To encrypt traffic between Cloudflare and DigitalOcean you will need to add a certificate to DigitalOcean's load balancer. This will be done at a later step, and at this point, you only need to create and download this certificate from Cloudflare. Go to your domain > SSL/TLS > Origin Server and click "Create Certificate". Don't change the Key Format from PEM. Copy the data from the "Origin Certificate" and save it as `origin_certificate.pem`. Copy the data from the "Private Key" and save it as `private_key.pem`.

At this point, we have configured everything needed in Cloudflare. We can move on to DigitalOcean.

### Importing Origin Certificate to DigitalOcean
Now as you have two `.pem` files on your machine, you can import the origin certificate into DigitalOcean.

```bash
doctl compute certificate create --type custom --name cloudsandboxcert --leaf-certificate-path origin_certificate.pem --private-key-path private_key.pem
```

### Creating DOKS Cluster
Create a DOKS cluster on DigitalOcean. For example, I'm creating a cluster with two worker nodes in Frankfurt. Here's the DOCTL command:
```bash
doctl kubernetes cluster create my-cluster --count 2 --size s-1vcpu-2gb
```

To manage the cluster, you will need to add an authentication token or certificate to your kubectl configuration file:
```bash
doctl kubernetes cluster kubeconfig save my-cluster
```

I'm running a sample application in Kubernetes with a Deployment of two Pods and a Service that is exposing this application through a managed DigitalOcean load balancer on ports 80 and 443. Here's the manifest file.

Add your origin certificate ID to `service.beta.kubernetes.io/do-loadbalancer-certificate-id`.
This is the certificate that we have imported in a previous step. You can use DOCTL to list the available certificates:
```bash
doctl compute certificate list
```

Note that allow rules in `service.beta.kubernetes.io/do-loadbalancer-allow-rules` only list Cloudflare's IP ranges. This means that the load balancer would only accept traffic coming from Cloudflare. Clients won't be able to circumvent Cloudflare by directly connecting to a load balancer's IP address on DigitalOcean. Up-to-date list of Cloudflare's IPs is available here: [Cloudflare IP Ranges](https://www.cloudflare.com/ips/)

You can create this application in Kubernetes by applying the manifest file:
```bash
kubectl apply -f manifest.yaml
```

After a few minutes check the Kubernetes Services:
```bash
kubectl get services -o

 wide
```

Note down the EXTERNAL-IP address. In Cloudflare navigate back to your domain > DNS > Records and add an A-record pointing to this address.

## Verification
- Traffic will get encrypted between client and Cloudflare.
- Traffic will get encrypted between Cloudflare and DigitalOcean load balancer.
- Traffic will not get encrypted between load balancer and Kubernetes worker nodes.
