# Istio Installation Guide

## Introduction

Istio is an open-source service mesh that provides a uniform way to connect, secure, control, and observe microservices. It helps you manage traffic flow across your services, enforce access policies, and collect telemetry data without requiring changes to your application code.

This guide covers the installation and basic configuration of Istio in your Kubernetes cluster.

## Prerequisites

Before installing Istio, ensure you have:

- A Kubernetes cluster running version 1.21 or higher
- Sufficient cluster privileges (cluster-admin)
- At least 2 vCPUs and 4 GB of memory available for Istio components
- kubectl installed and configured to communicate with your cluster

## Installation Methods

### Method 1: Using istioctl (Recommended)

1. **Download Istio**

   Download the latest Istio release:

   ```bash
   curl -L https://istio.io/downloadIstio | sh -
   ```

   Or download a specific version:

   ```bash
   curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.20.0 sh -
   ```

2. **Navigate to the Istio package directory**

   ```bash
   cd istio-1.20.0  # Replace with your version
   ```

   The installation directory contains:
   - Sample applications in `samples/` directory
   - The `istioctl` client binary in the `bin/` directory

3. **Add istioctl to your PATH**

   Linux or macOS:
   ```bash
   export PATH=$PWD/bin:$PATH
   ```

   Windows (PowerShell):
   ```powershell
   $env:PATH += ";$PWD\bin"
   ```

4. **Install Istio**

   For a default installation with a demo profile:
   ```bash
   istioctl install --set profile=demo -y
   ```

   Available profiles:
   - `default`: Recommended for production deployments
   - `demo`: Suitable for testing (includes more features)
   - `minimal`: Minimal set of components
   - `empty`: No components installed (base for custom installations)
   - `preview`: Latest experimental features

### Method 2: Using Helm

1. **Add the Istio Helm repository**

   ```bash
   helm repo add istio https://istio-release.storage.googleapis.com/charts
   helm repo update
   ```

2. **Create namespace for Istio**

   ```bash
   kubectl create namespace istio-system
   ```

3. **Install Istio base chart**

   ```bash
   helm install istio-base istio/base -n istio-system
   ```

4. **Install Istio discovery chart**

   ```bash
   helm install istiod istio/istiod -n istio-system --wait
   ```

5. **Install Istio ingress gateway (optional)**

   ```bash
   kubectl create namespace istio-ingress
   kubectl label namespace istio-ingress istio-injection=enabled
   helm install istio-ingress istio/gateway -n istio-ingress --wait
   ```

## Configuration

### Enable Automatic Sidecar Injection

To enable automatic sidecar injection for a namespace:

```bash
kubectl label namespace <namespace> istio-injection=enabled
```

Example:
```bash
kubectl label namespace default istio-injection=enabled
```

### Manual Sidecar Injection

For namespaces without automatic injection:

```bash
kubectl apply -f <(istioctl kube-inject -f your-app-deployment.yaml)
```

### Configure Istio with IstioOperator

Create an `IstioOperator` configuration file (e.g., `istio-config.yaml`):

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  profile: demo
  components:
    egressGateways:
    - name: istio-egressgateway
      enabled: true
  values:
    global:
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

Apply the configuration:

```bash
istioctl install -f istio-config.yaml
```

## Verification

### Verify Installation

Check if Istio components are deployed:

```bash
kubectl get pods -n istio-system
```

Verify the Istio version:

```bash
istioctl version
```

### Deploy Sample Application

Deploy the Bookinfo sample application:

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

Verify the deployment:

```bash
kubectl get services
kubectl get pods
```

Access the application:

```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
istioctl analyze
```

Get the ingress gateway URL:

```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
echo "http://$GATEWAY_URL/productpage"
```

## Uninstallation

### Remove Istio Applications

First, remove any applications that use Istio:

```bash
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
```

### Uninstall Istio

Using istioctl:

```bash
istioctl uninstall --purge -y
```

Or using Helm:

```bash
helm uninstall istio-ingress -n istio-ingress
helm uninstall istiod -n istio-system
helm uninstall istio-base -n istio-system
kubectl delete namespace istio-system
kubectl delete namespace istio-ingress
```

## Troubleshooting

### Common Issues

1. **Pods stuck in Pending state**
   - Check if your cluster has enough resources
   - Verify node taints and pod tolerations

2. **Sidecar injection not working**
   - Ensure the namespace is labeled correctly
   - Check if the pod has the `sidecar.istio.io/inject: "false"` annotation

3. **Connection issues between services**
   - Verify service and pod selectors match
   - Check Istio authentication and authorization policies
   - Use `istioctl analyze` to identify potential issues

### Debugging Tools

1. **Check proxy status**
   ```bash
   istioctl proxy-status
   ```

2. **Analyze configuration**
   ```bash
   istioctl analyze
   ```

3. **Debug envoy configuration**
   ```bash
   istioctl proxy-config all <pod-name>.<namespace>
   ```

4. **View logs**
   ```bash
   kubectl logs -n istio-system -l app=istiod
   kubectl logs <pod-name> -c istio-proxy
   ```

## References

- [Official Istio Documentation](https://istio.io/latest/docs/)
- [Istio GitHub Repository](https://github.com/istio/istio)
- [Istio Release Notes](https://istio.io/latest/news/releases/)
- [Istio Community](https://istio.io/latest/about/community/)
