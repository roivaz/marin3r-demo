
# marin3r demo

## Deploy a test application

Create the following Deployment in the `marin3r-demo` namespace.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuard
  namespace: marin3r-demo
  labels:
    app: kuard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kuard
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
        - name: kuard
          image: gcr.io/kuar-demo/kuard-amd64:blue
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
```

Create a service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kuard
  namespace: marin3r-demo
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: http
  selector:
    app: kuard
  type: LoadBalancer
```

## Create a certificate

Now lets create a Certificate, issued by Let's Encrypt:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: kuard
  namespace: marin3r-demo
spec:
  commonName: 'kuard.dev.3sca.net'
  dnsNames:
    - 'kuard.dev.3sca.net'
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-staging
  secretName: kuard
```

## Deploy marin3r

Install the marin3r-operator

Create a DiscoveryService in the `marin3r-demo` namespace.

## Modify the deployment to use marin3r

Edit the Deployment and add the following to the Pod template:

* The label `marin3r.3scale.net/status: "enabled"`
* The annotations `marin3r.3scale.net/node-id: kuard` and `marin3r.3scale.net/ports: https:8443`

Edit the Service's `spec.ports` with:

```yaml
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
```

## Add an EnvoyConfig object

The EnvoyConfig object holds all the config required to do TLS offloading in envoy and proxy pass requests to kuard's http port

```yaml
apiVersion: marin3r.3scale.net/v1alpha1
kind: EnvoyConfig
metadata:
  name: kuard
  namespace: marin3r-demo
spec:
  nodeID: kuard
  serialization: yaml
  envoyResources:
    secrets:
      - name: kuard.dev.3sca.net
        ref:
          name: kuard
          namespace: marin3r-demo
    clusters:
      - name: kuard
        value: |
          name: kuard
          connect_timeout: 2s
          type: STRICT_DNS
          dns_lookup_family: V4_ONLY
          lb_policy: ROUND_ROBIN
          load_assignment:
            cluster_name: kuard
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: 127.0.0.1
                          port_value: 8080
    listeners:
      - name: https
        value: |
          name: https
          address:
            socket_address:
              address: 0.0.0.0
              port_value: 8443
          filter_chains:
            - filters:
              - name: envoy.http_connection_manager
                typed_config:
                  "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
                  access_log:
                    - name: envoy.file_access_log
                      config:
                        path: /dev/stdout
                  stat_prefix: ingress_https
                  route_config:
                    name: local_route
                    virtual_hosts:
                      - name: kuard
                        domains: ["*"]
                        routes:
                          - match:
                              prefix: "/"
                            route:
                              cluster: kuard
                  http_filters:
                    - name: envoy.router
              transport_socket:
                name: envoy.transport_sockets.tls
                typed_config:
                  "@type": "type.googleapis.com/envoy.api.v2.auth.DownstreamTlsContext"
                  common_tls_context:
                    tls_certificate_sds_secret_configs:
                      - name: kuard.dev.3sca.net
                        sds_config:
                          ads: {}
```
