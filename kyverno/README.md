# Kyverno

## What is Kyverno?

> Kyverno is a policy engine designed for Kubernetes. It can validate, mutate, and generate configurations using admission controls and background scans. Kyverno policies are Kubernetes resources and do not require learning a new language. Kyverno is designed to work nicely with tools you already use like kubectl, kustomize, and Git.

## Installation

I installed the kyverno via the Helm:

```shell
# Add the Helm repository
helm repo add kyverno https://kyverno.github.io/kyverno/

# Scan your Helm repositories to fetch the latest available charts.
helm repo update

# Install the Kyverno Helm chart into a new namespace called "kyverno"
helm install kyverno --namespace kyverno kyverno/kyverno --create-namespace
```

The official documentation offers the following step to check the functionality:

Create the following policy:

```yaml
kubectl create -f- << EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: enforce
  rules:
  - name: check-for-labels
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "label 'app.kubernetes.io/name' is required"
      pattern:
        metadata:
          labels:
            app.kubernetes.io/name: "?*"
EOF
```

> It contains a single validation rule that requires that all Pods have a `app.kubernetes.io/name` label

Try creating a Deployment without the required label:

```shell
kubectl create deployment nginx --image=nginx
```

You should see an error:

```shell
Error from server: admission webhook "nirmata.kyverno.resource.validating-webhook" denied the request:

resource Deployment/default/nginx was blocked due to the following policies

require-labels:
  autogen-check-for-labels: 'Validation error: label `app.kubernetes.io/name` is required;
    Validation rule autogen-check-for-labels failed at path /spec/template/metadata/labels/app.kubernetes.io/name/'
```

Create a Pod with the required label:

```shell
kubectl run nginx --image nginx --labels app.kubernetes.io/name=nginx
```

And you will not see any errors.

Clean up by deleting all cluster policies:

```shell
kubectl delete cpol --all
```

## Network Policies

In this section, we want to create a policy that applies the specific network policies when a namespace is created.

Create the `kyverno-allow-ingress.yaml` that contains the following:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: allow-ingress-namespace
  annotations:
    policies.kyverno.io/title: Add Network Policy
    policies.kyverno.io/category: Multi-Tenancy
    policies.kyverno.io/subject: NetworkPolicy
    policies.kyverno.io/description: >-
      allow connection from ingress-nginx namespace
spec:
  rules:
  - name: ingress-allow
    match:
      resources:
        kinds:
        - Namespace
    generate:
      kind: NetworkPolicy
      name: allow-from-ingress
      namespace: "{{request.object.metadata.name}}"
      data:
        spec:
          podSelector:
            matchLabels:
          ingress:
          - from:
            - namespaceSelector:
                matchLabels:
                  name: ingress-nginx

```

To apply:

```shell
kubectl apply -f kyverno-allow-ingress.yaml
```

This policy allows the `ingress` network connection to the `ingress-nginx` namespace when a namespace is created.

Now for the same namespace's pod connectivity create the following yaml:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-all-namespace
  annotations:
    policies.kyverno.io/title: Add Network Policy
    policies.kyverno.io/category: Multi-Tenancy
    policies.kyverno.io/subject: NetworkPolicy
    policies.kyverno.io/description: >-
      allow connection between pods in same namesapce      
spec:
  rules:
  - name: default-deny
    match:
      resources:
        kinds:
        - Namespace
    generate:
      kind: NetworkPolicy
      name: allow-same-namespace
      namespace: "{{request.object.metadata.name}}"
      data:
        spec:
          podSelector:
            matchLabels:
          ingress:
          - from:
            - podSelector: {}
```

To apply:

```shell
kubectl apply -f kyverno-same-ns.yaml
```

**Note:** This policies only apply to the new namespace and does not affect current namespaces.

### Verification

To verify the functionality you need to create a new namespace, create a simple nginx with svc and try to connect to the svc from the other namespaces.

```shell
kubectl create namespace test
kubectl run nginx -n test --image=nginx
kubectl expose po nginx --port 80 -n test

```

```shell
kubectl run busybox -n wordpress --rm -ti --image=busybox -- /bin/sh ##wordpress could be any ns
wget nginx.test.svc ##wget the svc should not be connected
```

```shell
kubectl run busybox -n ingress-nginx --rm -ti --image=busybox -- /bin/sh 
wget nginx.test.svc ##wget the svc should be connected
```



## References

[Kyverno Github Page](https://github.com/kyverno/kyverno)

[Kyverno Installation Guide](https://kyverno.io/docs/installation/)

[Kyverno Network Policy Documantion](https://kyverno.io/policies/best-practices/add_network_policy/)

[Alternative Simple unrated Solution](https://github.com/flanksource/template-operator)