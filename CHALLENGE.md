# Considerations
This entire solution is designed to provide a GitOps-based environment that enables both standardization of deployment aspects and flexibility for engineers to develop their products as Helm charts. `ops-sample` is a good example showing how various teams can develop Helm charts that unify deployment procedures while maintaining flexibility.

## Anticipating Future Needs
- Authorization policies ([myapp/templates/minio/authorization.yaml](https://github.com/BrutalHex/ops-sample/blob/646c81900ca777c2c6f8aa503d48ab44de27e69f/myapp/templates/s3www/auth.yaml))
- `HTTPRoute`, the newer version of Ingress ([example](https://github.com/BrutalHex/ops-sample/blob/d07e5241057b3ee0d4d4f0214058e0737cfeca99/myapp/templates/s3www/http-route.yaml))

I do not consider adding `Prometheus` labelings a good idea. I follow the Principle of **Least Surprise**: anything we add for ports might change in the future. Also, it might mislead new project members into thinking a feature is active and in use while it is not.

## Limitations

### Problem: The Namespace resource should not be included in the chart
- **Purpose:** The intent was to enable Istio ambient mesh without involving the base infrastructure project.
- **Solution:** Install Kyverno in [ops-kubernetes-base](https://github.com/BrutalHex/ops-kubernetes-base) and add an automated patch for namespaces:
    ```yaml
    apiVersion: kyverno.io/v1
    kind: ClusterPolicy
    metadata:
      name: add-default-namespace-labels
    spec:
      rules:
        - name: add-istio-labels-to-namespace
          match:
            resources:
              kinds:
                - Namespace
          mutate:
            patchStrategicMerge:
              metadata:
                labels:
                  istio.io/dataplane-mode: "ambient"
                  istio.io/use-waypoint: "waypoint"
    ```

### Problem: Labels and Annotations are not merged from values
- **Purpose:** Reduce manual effort for this sample project.
- **Solution:** Labels can be merged for various resources using the following sample:
    ```yaml
    {{- $defaultLabels := dict \
        "app.kubernetes.io/name" "myapp" \
        "app.kubernetes.io/instance" .Release.Name \
        "app.kubernetes.io/managed-by" "Helm" }}

    {{- $customLabels := .Values.labels | default dict }}
    {{- $mergedLabels := merge $defaultLabels $customLabels }}

    metadata:
      labels:
    {{ toYaml $mergedLabels | indent 4 }}
    ```

### Problem: Use of tag instead of image hash
- **Purpose:** Improve reliability and reproducibility
- **Solution:** Remove tags from `values.yaml` and use image digests instead. For example:
    `image: {{ .Values.image.repository }}@{{ .Values.image.digest }}`
    To find the digest:
    ```sh
    docker pull nginx:1.25.2
    skopeo inspect docker://nginx:1.25.2 | jq .Digest
    ```
