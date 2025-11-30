# Microservices Ingress on EKS

This repo contains two demo services (`user` and `product`) and an NGINX-based `Ingress` resource that fronts both services inside an Amazon EKS cluster.

## Prerequisites

- AWS CLI configured with credentials allowed to manage the target EKS cluster.
- `kubectl` configured to talk to your cluster (`aws eks update-kubeconfig --name <cluster> --region <region>`).
- `helm` (v3) to install the NGINX Ingress Controller.
- The cluster must have worker nodes in public subnets (or private subnets with proper load balancer access) so the NGINX controller's `LoadBalancer` Service can get an external address.

## Deploy the demo services

```bash
kubectl apply -f user.yml
kubectl apply -f product.yml
```

Each manifest creates a Deployment and a ClusterIP Service on port 80.

## Install the NGINX Ingress Controller

If your cluster already has NGINX Ingress installed, skip this section. Otherwise:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace --set controller.service.type=LoadBalancer --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb
```

Wait for the controller Service to obtain an external hostname:

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

The `EXTERNAL-IP` (or `HOSTNAME`) will ultimately be how you reach the microservices.

## Apply the Ingress

```bash
kubectl apply -f ingress.yml
```

Key points in `ingress.yml`:

- `ingressClassName: nginx` (and matching annotation) ensures the manifest is reconciled by the NGINX controller.
- `nginx.ingress.kubernetes.io/rewrite-target: /` rewrites `/users` and `/products` requests to `/` for the backend NGINX containers.

Verify the ingress status:

```bash
kubectl get ingress microservice-ingress
kubectl describe ingress microservice-ingress
```

Once the ingress is ready, you can hit the controller's external address:

```bash
curl http://<nginx-controller-hostname>/users
curl http://<nginx-controller-hostname>/products
```

If you need a friendly DNS name, create a Route53 record that CNAMEs to the controller's hostname.

## Troubleshooting

- Ensure the `ingress-nginx-controller` pods are running (`kubectl get pods -n ingress-nginx`).
- Check events on the ingress (`kubectl describe ingress microservice-ingress`) for path or backend errors.
- Confirm the services are reachable inside the cluster (`kubectl port-forward svc/user-service 8080:80`).

## Cleanup

```bash
kubectl delete -f ingress.yml
kubectl delete -f user.yml
kubectl delete -f product.yml
helm uninstall ingress-nginx -n ingress-nginx
```

This removes the demo workloads and the ingress controller from your EKS cluster.
