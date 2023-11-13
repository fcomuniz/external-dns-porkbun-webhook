# external-dns-netcup-webhook

External-DNS Webhook Provider to manage Netcup DNS Records


> [!WARNING]
> Completely untested code. Might eat your DNS records. You have been warned.


## Setting up external-dns for Netcup

This tutorial describes how to setup external-dns for usage within a Kubernetes cluster using Netcup as the domain provider.

Make sure to use external-dns version 0.14.0 or later for this tutorial.

### Creating Netcup Credentials

A secret containing the a Netcup API token and an API Password is needed for this provider. You can get a token for your user [here](https://www.customercontrolpanel.de/daten_aendern.php?sprung=api).

To create the API token secret you can run `kubectl create secret generic netcup-api-key --from-literal=NETCUP_API_KEY=<replace-with-your-access-token>`.

To create the API password secret you can run `kubectl create secret generic netcup-api-password --from-literal=NETCUP_API_PASSWORD=<replace-with-your-access-token>`.

### Deploy external-dns

Connect your `kubectl` client to the cluster you want to test external-dns with.

Besides the API key and password, it is mandatory to provide a customer id as well as a list of DNS zones you want external-dns to manage. The hosted DNS zones will be provides via the `--domain-filter`.

Then apply one of the following manifests file to deploy external-dns.

### Manifest (for clusters without RBAC enabled)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:latest
        args:
        - --source=ingress
        - --source=service
        - --provider=webhook
      - name: external-dns-webhook-provider
        image: ghcr.io/mrueg/external-dns-netcup-webhook:latest
        args:
        - --domain-filter="example.com"
        - --netcup-customer-id="YOUR_CUSTOMER_ID"
        env:
        - name: NETCUP_API_KEY
          valueFrom:
            secretKeyRef:
              key: NETCUP_API_KEY
              name: netcup-api-key
        - name: NETCUP_API_PASSWORD
          valueFrom:
            secretKeyRef:
              key: NETCUP_API_PASSWORD
              name: netcup-api-password

```

### Deploying an Nginx Service

Create a service file called 'nginx.yaml' with the following contents:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: example.com
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Note the annotation on the service; use the same hostname as the Netcup DNS zone created above. The annotation may also be a subdomain
of the DNS zone (e.g. 'www.example.com').

By setting the TTL annotation on the service, you have to pass a valid TTL, which must be 120 or above.
This annotation is optional, if you won't set it, it will be 1 (automatic) which is 300.

external-dns uses this annotation to determine what services should be registered with DNS.  Removing the annotation
will cause external-dns to remove the corresponding DNS records.

Create the deployment and service:

```
$ kubectl create -f nginx.yaml
```

Depending where you run your service it can take a little while for your cloud provider to create an external IP for the service.

Once the service has an external IP assigned, external-dns will notice the new service IP address and synchronize
the Netcup DNS records.

### Verifying Netcup DNS records

Check your [Netcup domain overview](https://www.customercontrolpanel.de/domains.php) to view the domains associated with your Netcup account. There you can view the records for each domain.

The records should show the external IP address of the service as the A record for your domain.

### Cleanup

Now that we have verified that external-dns will automatically manage Netcup DNS records, we can delete the tutorial's example:

```
$ kubectl delete -f nginx.yaml
$ kubectl delete -f externaldns.yaml
