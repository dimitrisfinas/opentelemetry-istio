# HOW TO INSTALL ISTIO WITH OPENTELEMETRY/LIGHTSTEP

## VERSION TESTED
- Istio v1.16.2
- Kubernetes v1.23.14
- Helm 3.6 or above

## PREREQUISITES
- kubectl
- helm


## INSTALL

1. Configure the Helm repository
```shell
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

2. Create a namespace istio-system for Istio components
```shell
kubectl create namespace istio-system
```

3. Install the Istio base chart which contains cluster-wide resources used by the Istio control plane
```shell
helm install istio-base istio/base -n istio-system
```

4. Install the Istio discovery chart which deploys the istiod service
```shell
helm install istiod istio/istiod -n istio-system --wait
```

5. Add the ingress gateway
```shell
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
helm install istio-ingress istio/gateway -n istio-ingress --wait
```
> **_NOTE:_** The namespace the gateway is deployed in must not have a istio-injection=disabled label.

6. Add OpenTelemetry Collector
  - Create OpenTelemetry Collector deployment files like example below
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: opentelemetry-collector-conf
  namespace: istio-system
  labels:
    app: opentelemetry-collector
data:
  opentelemetry-collector-config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:
      batch:
    exporters:
      otlp/lightstep:
        endpoint: ingest.lightstep.com:443
        headers:
          "lightstep-access-token": "<YOUR_TOKEN>"
      logging:
        loglevel: debug
    extensions:
      health_check:
    service:
      extensions:
      - health_check
      pipelines:
        logs:
          receivers: [otlp]
          processors: [batch]
          exporters: [logging]
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [logging, otlp/lightstep]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [logging, otlp/lightstep]
---
apiVersion: v1
kind: Service
metadata:
  name: opentelemetry-collector
  namespace: istio-system
  labels:
    app: opentelemetry-collector
spec:
  ports:
    - name: grpc-otlp # Default endpoint for OpenTelemetry receiver.
      port: 4317
      protocol: TCP
      targetPort: 4317
  selector:
    app: opentelemetry-collector
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opentelemetry-collector
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: opentelemetry-collector
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: opentelemetry-collector
        sidecar.istio.io/inject: "false" # do not inject
    spec:
      containers:
        - command:
            - "/otelcol"
            - "--config=/conf/opentelemetry-collector-config.yaml"
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
          image: otel/opentelemetry-collector:0.71.0
          imagePullPolicy: IfNotPresent
          name: opentelemetry-collector
          ports:
            - containerPort: 4317
              protocol: TCP
          resources:
            limits:
              cpu: "2"
              memory: 4Gi
            requests:
              cpu: 200m
              memory: 400Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
            - name: opentelemetry-collector-config-vol
              mountPath: /conf
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      terminationGracePeriodSeconds: 30
      volumes:
        - configMap:
            defaultMode: 420
            items:
              - key: opentelemetry-collector-config
                path: opentelemetry-collector-config.yaml
            name: opentelemetry-collector-conf
          name: opentelemetry-collector-config-vol
```

  - deploy it
```shell
kubectl apply -f otel.yaml -n istio-system
```

7. Add Istio Telemetry API
  - Define OpenTelemetry provider to use and configure mesh default behavior by creating file `istio-telemetry.yaml` with content below
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio
  namespace: istio-system
  annotations:
    meta.helm.sh/release-name: istiod
    meta.helm.sh/release-namespace: istio-system
  labels:
    app.kubernetes.io/managed-by: Helm
    install.operator.istio.io/owning-resource: unknown
    istio.io/rev: default
    operator.istio.io/component: Pilot
    release: istiod
data:
  meshNetworks: |-
    networks: {}

  mesh: |-
    defaultConfig:
      discoveryAddress: istiod.istio-system.svc:15012
    defaultProviders:
      tracing:
        - opentelemetry
    extensionProviders:
      - name: "opentelemetry"
        opentelemetry:
          service: "opentelemetry-collector.istio-system.svc.cluster.local"
          port: 4317
    enablePrometheusMerge: true
    rootNamespace: null
    trustDomain: cluster.local

---
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  accessLogging:
    - providers:
      - name: opentelemetry
  tracing:
  - providers:
    - name: opentelemetry
    customTags:
      test:
        literal:
          value: "this is a test of custom attribute"
    randomSamplingPercentage: 100
```

  - deploy this configuration
```shell
kubectl apply -f istio-telemetry.yaml -n istio-system
```


8. Deploy Demo Application
  - Enable sidecar injection
```shell
kubectl label namespace default istio-injection=enabled
```

  - download Istio package with demo application
```shell
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.16.2 TARGET_ARCH=x86_64 sh -
```

  - deploy the bookinfo application
```shell
cd istio-1.16.2
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

- Add it to the ingress gateway
    - edit file `./samples/bookinfo/networking/bookinfo-gateway.yaml`
    - replace `selector: istio: ingressgateway` by `selector: istio: ingress`

```shell
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```


## VERIFY

- Status of the installation can be verified using Helm
```shell
helm status istiod -n istio-system
```

- List all the Istio charts installed in istio-system namespace:
```shell
helm ls -n istio-system
```
       - Result: `istio-base, istiod`

- List demo application is there:
```shell
kubectl get services
```
      - Result: `details, kubernetes, productpage, ratings, reviews`

```shell
kubectl get pods
```
      - Result: `details-v1, productpage-v1, ratings-v1, reviews-v1, reviews-v2, reviews-v3`

- Test the application from command line
```shell
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```
      - Result: `<title>Simple Bookstore App</title>`

- Test the application from browser
    - go to `http://<EXTERNAL-IP>:80/productpage`
    - where you can get EXTERNAL-IP from command:
```shell
kubectl get svc istio-ingress -n istio-ingress
```


## REMOVE
1. Delete any Istio gateway chart installations
```shell
helm delete istio-ingress -n istio-ingress
kubectl delete namespace istio-ingress
```

2. Delete Istio discovery chart:
```shell
helm delete istiod -n istio-system
```

3. Delete Istio base chart:
```shell
helm delete istio-base -n istio-system
```

4. Delete the istio-system namespace:
```shell
kubectl delete namespace istio-system
```

5. Delete OpenTelemetry collector
```shell
kubectl delete -f otel.yaml -n istio-system
```

6. Delete Demo application
```shell
cd istio-1.16.2
kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
```

## SOURCE
- Istio Kubernetes versions compatibility matrix https://istio.io/latest/docs/releases/supported-releases/#support-status-of-istio-releases

- Istio helm installation: https://istio.io/latest/docs/setup/install/helm/

- Istio Telemetry API: https://istio.io/latest/docs/tasks/observability/telemetry/

- Istio sidecar configuration: https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#controlling-the-injection-policy

- Collector for Istio: https://github.com/istio/istio/tree/master/samples/open-telemetry

- Deploying demo application
https://istio.io/latest/docs/setup/getting-started/

## TROUBLESHOOT

- getting error `warn	Not able to configure requested tracing provider "opentelemetry": could not find cluster for tracing provider "envoy.tracers.opentelemetry": could not find service opentelemetry-collector in Istio service registry"`
    - either you're calling an external service or your collector service URL is not correct
    - check your provider service value in your `istio-telemetry.yaml` file
    - it should be something like `<service-name>.<namespace>.svc.cluster.local`, example: `opentelemetry-collector.istio-system.svc.cluster.local`


- getting `HTTP 404` when connecting to demo application
    - this may be because name of istio ingress gateway has changed between versions
    - be sure to update the selector of your file `./samples/bookinfo/networking/bookinfo-gateway.yaml`
    - replace `selector: istio: ingressgateway` by `selector: istio: ingress`
    - re-apply the file
