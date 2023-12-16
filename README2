```
# Integrating DigitalOcean Kubernetes with Cloudflare for Enhanced Security

## Problem Statement
While DigitalOcean's managed Kubernetes service (DOKS) provides basic DDoS protection, applications may require advanced DDoS protection, a Web Application Firewall (WAF), and other security enhancements. Cloudflare offers these services, and this tutorial will guide you through integrating Cloudflare with DOKS for enhanced security.

## Architecture
The architecture involves a web application running in DOKS:
- A Kubernetes Deployment manages the application.
- A Service exposes the application via DigitalOcean's managed load balancer.
- Client requests go to Cloudflare via HTTPS.
- Cloudflare inspects requests and filters out malicious traffic.
- Legitimate requests are proxied to DigitalOcean over HTTPS.
- DigitalOcean's load balancer terminates TLS and forwards requests over HTTP to worker nodes.
- The centralized management of TLS certificates at the load balancer level simplifies certificate management.

## Configuration Prerequisites
Before starting, ensure these tools are installed and configured:
- [doctl](https://docs.digitalocean.com/reference/doctl/how-to/install/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [jq](https://stedolan.github.io/jq/download/)

## Configuration

### Adding a Site to Cloudflare
1. Register your domain name.
2. Add your domain to Cloudflare, which will manage its DNS records.
3. Update your domain's nameservers at your registrar (e.g., GoDaddy, Namecheap) to point to Cloudflare.

### Enabling HTTPS End-to-End
#### Configure Edge Certificates
In the Cloudflare control panel:
1. Navigate to your domain > SSL/TLS > Edge Certificates.
2. Enable "Always Use HTTPS" for consistent encryption between the client and Cloudflare.

#### Set Up Full (strict) Encryption Mode
1. Go to SSL/TLS > Overview.
2. Change the encryption mode to "Full (strict)" for encryption between Cloudflare and DigitalOcean.

#### Prepare Certificates for DigitalOcean
1. Under SSL/TLS > Origin Server, click "Create Certificate".
2. Download the certificate and key, saving them as `origin_certificate.pem` and `private_key.pem`.

### Importing Origin Certificate to DigitalOcean
Import the origin certificate with:
```bash
doctl compute certificate create --type custom --name cloudsandboxcert --leaf-certificate-path origin_certificate.pem --private-key-path private_key.pem
```

### Creating DOKS Cluster
Create a cluster using `doctl`:
```bash
doctl kubernetes cluster create my-cluster --count 2 --size s-1vcpu-2gb
```
Add the cluster to your kubectl configuration:
```bash
doctl kubernetes cluster kubeconfig save my-cluster
```

### Kubernetes Application Deployment
Deploy an application with the following manifest:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deployment
# ... (remaining YAML content)

kind: Service
metadata:
  name: hello-world-service
  # Annotations for configuring the load balancer
  # ... (remaining annotations)
# ... (remaining YAML content)
```

Link the imported certificate to the load balancer:
```bash
doctl compute certificate list
```
Update `service.beta.kubernetes.io/do-loadbalancer-certificate-id` with your certificate ID.

Set load balancer to only accept traffic from Cloudflare:
- Refer to [Cloudflare IP Ranges](https://www.cloudflare.com/ips/) for the updated IP list.

Deploy the application:
```bash
kubectl apply -f manifest.yaml
```

Check the Services after deployment:
```bash
kubectl get services -o wide
```
In Cloudflare, add an A-record pointing to the EXTERNAL-IP address.

## Verification
Verify the setup using `curl` against the Kubernetes echo server. Look for the following headers in the response to confirm the traffic flow:
```json
"headers": {
  "host": "cloudsandbox.online",
  "x-forwarded-for": "32.185.143.12,172.71.103.51",
  "cf-ray": "835fdzzc4a665e-AMS",
  "x-forwarded-proto": "https",
  "cf-connecting-ip": "32.185.143.12",
  "cdn-loop": "cloudflare"
}
```
- The headers confirm encrypted traffic between the client and Cloudflare, and between Cloudflare and DigitalOcean.
- Traffic from the load balancer to worker nodes is not encrypted.
```
