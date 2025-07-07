# Introduction
The ops-sample project is web interface to serve a simple object storage based [`Minio`](https://min.io) helm chart.
The chart repository address is [`https://brutalhex.github.io/ops-sample/`](https://brutalhex.github.io/ops-sample/).
For more details, refer to the [ops-infra](https://github.com/BrutalHex/ops-infra) repository.

# Release
This project creates a basic Helm repository using GitHub Pages.
To release a new version of the website:  
- Modify the contents of the `ops-sample/myapp/templates` folder  
- Increase the chart version  
- Push the code  
- Execute [Publish Helm Chart](https://github.com/BrutalHex/ops-sample/actions/workflows/release.yaml) in GitHub Actions.

# Installation
This project is designed to work alongside the [ops-kubernetes-base](https://github.com/BrutalHex/ops-kubernetes-base) repository and should not be installed without it. To test the workloads on a local Kubernetes cluster, follow the instructions below.

> **Note:** Make sure the Istio service mesh is installed and properly configured in your Kubernetes cluster to enable all features of this chart before deploying it. However,it work without Istio workloads too.

To get started quickly, you can use [kind](https://kind.sigs.k8s.io/) to create a local Kubernetes cluster:

1. **Create a kind cluster:**
   ```sh
   kind create cluster --name myapp-demo
   ```

2. **Install Istio CRDs and base components:**
   ```sh
   helm install istio-base base --repo https://istio-release.storage.googleapis.com/charts --namespace istio-system --create-namespace
   ```
3. **Install Gateway-api CRDs:**
    ```sh
    kubectl apply -k "https://github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.0.0"
    ```
4. **Add the Helm repository:**
   ```sh
   helm repo add ops-sample https://brutalhex.github.io/ops-sample/
   helm repo update
   ```

5. **Create a custom `values.yaml` file (optional):**
   You can customize the chart by creating a `values.yaml` file. Below is a sample configuration:

    ```yaml
    gateway:
      # This is the Istio Gateway configuration for the application.
      namespace: istio-system
      # The name of the Gateway resource.
      name: waypoint
      # The URL that the Gateway will serve.
      url: example.com

    s3www:
      nodeSelector: {}
      tolerations: []
      affinity: {}
      replicaCount: 1
      image:
        repository: y4m4/s3www
        tag: latest
      resources:
        requests:
          cpu: 50m
          memory: 200Mi
        limits:
          cpu: 100m
          memory: 300Mi
      deployment:
        labels: {}
        annotations:
          description: s3www serving static bucket content
      podLabels: {}
      podAnnotations: {}
      service:
        port: 8080
        targetPort: 8080

    minio:
      image:
        repository: quay.io/minio/minio
        tag: latest
      resources:
        requests:
          memory: 1Gi
          cpu: 1
        limits:
          memory: 1Gi
          cpu: 1
      replicas: 1
      mode: standalone
      serviceName: myapp-minio
      ## Disable TLS, the mTLS provides secure communication.
      tls:
        enabled: false
      existingSecret: myapp-minio-credentials
      buckets:
        - name: web
          policy: readwrite
          purge: false
          versioning: false
          objectlocking: false
      users: []
      consoleService:
        type: ClusterIP
        port: "9001"
      persistence:
        enabled: true
        size: 1Gi
    ```

6. **Install the chart from the repository:**
    >
    > ⚠️ **Note:** You can safely ignore the following error during this step:
    >
    > Error: INSTALLATION FAILED: 1 error occurred:
    >         * namespaces "test" already exists
    >
    >     
     ```sh
     helm install myapp ops-sample/myapp -f values.yaml --namespace test --create-namespace
     ```    
     Or, to install local chart from source code for development:
     ```sh
     helm install myapp ./ops-sample/myapp -f values.yaml --namespace test --create-namespace
     ```
7. **Wait for healthy status of Minio and s3www**

8. **Upgrade the chart (if needed):**
   ```sh
   helm upgrade myapp ops-sample/myapp -f values.yaml
   # or for local chart
   helm upgrade myapp ./ops-sample/myapp -f values.yaml
   ```

9. **Delete the kind cluster when done:**
   ```sh
   kind delete cluster --name myapp-demo
   ```

