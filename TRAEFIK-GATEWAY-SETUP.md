# Traefik Gateway API Setup Instructions

This document provides instructions for setting up Traefik Ingress Controller with Gateway API support to use with the AKS Store Demo application.

## Overview

The `aks-store-ingress-quickstart.yaml` file has been updated to use the Kubernetes Gateway API with Traefik instead of the traditional nginx-based Ingress controller. The Gateway API provides a more flexible and standardized way to configure ingress traffic.

## Prerequisites

Before deploying the application with Traefik Gateway API support, you need to:

1. Have a Kubernetes cluster running (AKS, Kind, Minikube, etc.)
2. Have `kubectl` configured to access your cluster
3. Have `helm` installed (for Traefik installation)

## Installation Steps

### Step 1: Install Gateway API CRDs

The Gateway API requires Custom Resource Definitions (CRDs) to be installed in your cluster:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

Verify the CRDs are installed:

```bash
kubectl get crd | grep gateway
```

You should see resources like `gatewayclasses.gateway.networking.k8s.io`, `gateways.gateway.networking.k8s.io`, and `httproutes.gateway.networking.k8s.io`.

### Step 2: Install Traefik with Gateway API Support

Add the Traefik Helm repository and install Traefik with Gateway API provider enabled:

```bash
# Add Traefik Helm repository
helm repo add traefik https://traefik.github.io/charts
helm repo update

# Install Traefik with Gateway API provider enabled
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set providers.kubernetesGateway.enabled=true
```

For development/testing environments (like Kind or Minikube), you may want to use NodePort:

```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set ports.web.nodePort=30080 \
  --set ports.websecure.nodePort=30443 \
  --set providers.kubernetesGateway.enabled=true
```

### Step 3: Verify Traefik Installation

Check that Traefik is running:

```bash
kubectl get pods -n traefik
kubectl get svc -n traefik
```

### Step 4: Deploy the AKS Store Demo Application

Deploy the application using the updated manifest:

```bash
kubectl apply -f aks-store-ingress-quickstart.yaml
```

This will create:
- All application services (store-front, order-service, product-service, rabbitmq)
- A `GatewayClass` resource named "traefik"
- A `Gateway` resource named "store-front-gateway"
- An `HTTPRoute` resource named "store-front" that routes traffic to the store-front service

### Step 5: Access the Application

Get the external IP or hostname of the Gateway:

```bash
kubectl get gateway store-front-gateway
```

For LoadBalancer services (AKS, EKS, GKE), access the application at:
```
http://<GATEWAY_EXTERNAL_IP>
```

For NodePort services (Kind, Minikube), access the application at:
```
http://<NODE_IP>:30080
```

## Gateway API Resources Explained

### GatewayClass
Defines the controller that will handle Gateway resources. In our case, it's configured to use `traefik.io/gateway-controller`.

### Gateway
Defines the load balancer configuration, including:
- Which GatewayClass to use
- Listeners (HTTP on port 80)
- Allowed routes (from the same namespace)

### HTTPRoute
Defines the HTTP routing rules, replacing the traditional Ingress resource:
- Routes HTTP traffic from the Gateway to the store-front service
- Matches all paths with prefix "/"
- Forwards traffic to the store-front service on port 80

## Migration from nginx Ingress

The previous configuration used:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-front
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules: ...
```

This has been replaced with three Gateway API resources:
1. **GatewayClass**: Defines Traefik as the controller
2. **Gateway**: Defines the load balancer and listeners
3. **HTTPRoute**: Defines the routing rules

## Troubleshooting

### Gateway is not getting an external IP

Check the Gateway status:
```bash
kubectl describe gateway store-front-gateway
```

Check Traefik logs:
```bash
kubectl logs -n traefik -l app.kubernetes.io/name=traefik
```

### Application is not accessible

Verify all resources are created:
```bash
kubectl get gatewayclass
kubectl get gateway
kubectl get httproute
kubectl get svc store-front
```

Check the HTTPRoute status:
```bash
kubectl describe httproute store-front
```

### Traefik is not picking up the Gateway

Ensure the Gateway API provider is enabled in Traefik:
```bash
kubectl get deployment -n traefik traefik -o yaml | grep -A 5 providers
```

## Additional Resources

- [Kubernetes Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [Traefik Gateway API Documentation](https://doc.traefik.io/traefik/providers/kubernetes-gateway/)
- [Gateway API Implementations](https://gateway-api.sigs.k8s.io/implementations/)

