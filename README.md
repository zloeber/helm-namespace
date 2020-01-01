# helm-namespace

A generic helm3 namespace chart for use with helmfile or similar helm gluing toolsets. This is just a carry over solution for helm 3's inabilty to create namespaces for a release (likely going to change with helm 3.1). See (this thread)[https://github.com/roboll/helmfile/issues/891.] for more information and other options.

## Values

```
namespaces:
- namespace1
- namespace2
annotations:
  certmanager.k8s.io/disable-validation: true
labels:
  additional_label1: myvalue
```
## Example - helmfile

A simple example helmfile that creates a namespace as part of a cert-manager deployment. The default helm resource policy of 'keep' is used so that the namespace will not be removed in a helm destroy operation. Default tillerless plugin options are also set if this helmfile is created with helm version 2. I only include the namespace generation in this example for brevity.

```
helmDefaults:
  tillerless: true
  tillerNamespace: platform
  atomic: false
  verify: false
  wait: true
  timeout: 1200
  recreatePods: true
  force: true

repositories:
- name: jetstack
  url: "https://charts.jetstack.io"
- name: "incubator"
  url: "https://kubernetes-charts-incubator.storage.googleapis.com"
- name: "zloeber"
  url: "git+https://github.com/zloeber/helm-namespace@chart"
releases:
###############################################################################
## CERT-MANAGER - Automatic Let's Encrypt for Ingress  ########################
##   Also provides local CA for issuing locally valid TLS certificates  #######
###############################################################################
# References:
# - https://github.com/jetstack/cert-manager/blob/v0.11.0/deploy/charts/cert-manager/values.yaml
# Instructions for installing and testing correct install are at
# - https://docs.cert-manager.io/en/release-0.9/getting-started/install/kubernetes.html
- name: namespace-cert-manager
  # Helm 3 needs to put deployment info into a namespace. As this creates a namespace it will not exist yet so we use 'kube-system' 
  #  which should exist in all clusters.
  chart: "../charts/custom/namespace"
  namespace: kube-system
  labels:
    chart: namespace-cert-manager
    component: "cert-manager"
    namespace: "cert-manager"
  values:
  - namespaces:
    - cert-manager
    annotations:
      certmanager.k8s.io/disable-validation: true
- name: "cert-manager"
  namespace: "cert-manager"
  labels:
    chart: "cert-manager"
    repo: "stable"
    component: "kiam"
    namespace: "cert-manager"
    vendor: "jetstack"
    default: "false"
  chart: "jetstack/cert-manager"
  version: "v0.9.0"
  wait: true
  installed: {{ env "CERT_MANAGER_INSTALLED" | default "true" }}
  hooks:
    # This hoook adds the CRDs
    - events: ["presync"]
      showlogs: true
      command: "/bin/sh"
      args: ["-c", "kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/deploy/manifests/00-crds.yaml"]
  values:
    - fullnameOverride: cert-manager
      rbac:
        create: {{ env "RBAC_ENABLED" | default "true" }}
      ingressShim:
        defaultIssuerName: '{{ env "CERT_MANAGER_INGRESS_SHIM_DEFAULT_ISSUER_NAME" | default "letsencrypt-staging" }}'
        ### Optional: CERT_MANAGER_INGRESS_SHIM_DEFAULT_ISSUER_KIND;
        defaultIssuerKind: '{{ env "CERT_MANAGER_INGRESS_SHIM_DEFAULT_ISSUER_KIND" | default "ClusterIssuer" }}'
{{ if env "CERT_MANAGER_IAM_ROLE" | default "" }}
      podAnnotations:
        iam.amazonaws.com/role: '{{ env "CERT_MANAGER_IAM_ROLE" }}'
{{ end }}
      serviceAccount:
        create: {{ env "RBAC_ENABLED" | default "true" }}
        name: '{{ env "CERT_MANAGER_SERVICE_ACCOUNT_NAME" | default "" }}'
{{- if eq (env "MONITORING_ENABLED" | default "true") "true" }}
      prometheus:
        enabled: true
        servicemonitor:
          enabled: true
          prometheusInstance: {{ env "PROMETHEUS_INSTANCE" | default "kube-prometheus" }}
          targetPort: 9402
          path: /metrics
          interval: 60s
          scrapeTimeout: 30s
{{ end }}
      webhook:
        enabled: false
      cainjector:
        enabled: true
      resources:
        limits:
          cpu: "200m"
          memory: "256Mi"
        requests:
          cpu: "50m"
          memory: "128Mi"
- name: 'cert-manager-issuers'
  chart: "incubator/raw"
  namespace: "cert-manager"
  labels:
    component: "iam"
    namespace: "cert-manager"
    default: "true"
  version: "0.1.0"
  wait: true
  force: true
  recreatePods: true
  installed: {{ env "CERT_MANAGER_INSTALLED" | default "true" }}
  values:
  - resources:
    - apiVersion: certmanager.k8s.io/v1alpha1
      kind: ClusterIssuer
      metadata:
        name: letsencrypt-staging
      spec:
        acme:
          server: https://acme-staging-v02.api.letsencrypt.org/directory
          email: {{ coalesce (env "SMTP_RECIPIENT") (env "CERT_MANAGER_EMAIL") (env "KUBE_LEGO_EMAIL") "user@example.com" }}
          privateKeySecretRef:
            name: letsencrypt-staging
          solvers:
            - http01:
                ingress:
                  class: nginx
{{- if env "CERT_MANAGER_IAM_ROLE" | default "" }}
            # Enable the DNS-01 challenge provider
            - dns01:
                route53: {}
{{- end }}
    - apiVersion: certmanager.k8s.io/v1alpha1
      kind: ClusterIssuer
      metadata:
        name: letsencrypt-prod
      spec:
        acme:
          server: https://acme-v02.api.letsencrypt.org/directory
          email: {{ coalesce (env "SMTP_RECIPIENT") (env "CERT_MANAGER_EMAIL") (env "KUBE_LEGO_EMAIL") "user@example.com" }}
          privateKeySecretRef:
            name: letsencrypt-prod
          solvers:
            - http01:
                ingress:
                  class: nginx
{{- if env "CERT_MANAGER_IAM_ROLE" | default "" }}
            # Enable the DNS-01 challenge provider
            - dns01:
                route53: {}
{{- end }}

```
