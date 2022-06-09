---
title: "Secure Supply chain"
linkTitle: "Secure supply chain"
weight: 3
date: 2017-01-05
description: >
  How to gate node upgrades in Kubernetes with Kubewarden
---

This example will show how it is possible to gate upgrades to a cluster with 

# Pre-req 

- A kubernetes cluster built with Elemental-toolkit
- Kubewarden and System upgrade controller installed

## Install Kubewarden

Kubewarden needs cert
### 1) Cert manager

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --for=condition=Available deployment --timeout=2m -n cert-manager --all
```

### 2) Kubewarden

```bash
helm repo add kubewarden https://charts.kubewarden.io

helm install --wait -n kubewarden --create-namespace kubewarden-crds kubewarden/kubewarden-crds

helm install --wait -n kubewarden kubewarden-controller kubewarden/kubewarden-controller

helm install --wait -n kubewarden kubewarden-defaults kubewarden/kubewarden-defaults
```

## Cosign validation Policy

# Accept only signed by master workflow

```bash
kubectl apply -f - <<EOF
apiVersion: policies.kubewarden.io/v1alpha2
kind: AdmissionPolicy
metadata:
  name: gate-upgrades
  namespace: system-upgrade
spec:
  module: registry://ghcr.io/kubewarden/policies/verify-image-signatures:v0.1.3
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
    operations:
    - CREATE
    - UPDATE
  mutating: false
  settings:
    modifyImagesWithDigest: true
    signatures:
    - image: quay.io/c3os
      keyless:
      - issuer: https://token.actions.githubusercontent.com
        subject: https://github.com/c3os-io/c3os/.github/workflows/image.yaml@refs/heads/master

EOF
```

Notes: subject/issuer have to match 1:1
COSIGN_EXPERIMENTAL=1 cosign verify quay.io/c3os/c3os:opensuse-latest | jq -r '.[0].optional.Subject'

In this example, quay.io/c3os/c3os:opensuse-latest is signed, while quay.io/c3os/c3os:opensuse-v1.23.6-53 is not.

# 

indeed if we try to apply an ugprade to a tagged version in policy server logs, can see error lines of validation failing

```
$ policy_pod=$(kubectl get pods -n kubewarden -l app=kubewarden-policy-server-default -o jsonpath='{.items[0].metadata.name}')
$ kubectl logs -n kubewarden $policy_pod
2022-06-09T09:12:42.972860Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-j9mch" namespace="system-upgrade" operation="CREATE" request_uid="3d9c9fa0-0d42-43b0-83ac-426fd2ec3bb8" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T09:13:03.148375Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-q4gnn" namespace="system-upgrade" operation="CREATE" request_uid="39269d6e-a351-476d-8d68-9448b7375db4" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T09:13:43.585755Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-7xk6b" namespace="system-upgrade" operation="CREATE" request_uid="3a63518e-0e93-4fe4-9fcf-0bf83f0d5080" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T09:15:04.077763Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-xkcdj" namespace="system-upgrade" operation="CREATE" request_uid="3a1152e5-bd4f-4f00-a9e1-be0b94c7e338" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T09:17:46.078840Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-bwzhv" namespace="system-upgrade" operation="CREATE" request_uid="98e391e6-962e-4635-912f-e6fe43079fb1" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T09:23:07.752249Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-bkfch" namespace="system-upgrade" operation="CREATE" request_uid="5f41c50a-4cc5-4e35-83a0-0a73cf345667" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T09:29:09.403549Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-nn8m9" namespace="system-upgrade" operation="CREATE" request_uid="5d30786b-4ae3-4cb4-b8cb-ee5400a133ae" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T09:35:10.980934Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-pq57f" namespace="system-upgrade" operation="CREATE" request_uid="bf1fc235-d68b-4435-a0e7-6d2e36fa0793" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T09:41:12.660548Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-wt5df" namespace="system-upgrade" operation="CREATE" request_uid="6a9eefba-41f6-4bfa-8b30-c4de004fbf86" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T09:47:14.282229Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-cr7ll" namespace="system-upgrade" operation="CREATE" request_uid="f94c4a75-bb0f-49a2-9fde-fe001b7947ca" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T09:53:16.018023Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-gxfxg" namespace="system-upgrade" operation="CREATE" request_uid="ca2fb400-b88a-452f-9b18-9b1c35962f4b" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T09:59:17.373130Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-t92vn" namespace="system-upgrade" operation="CREATE" request_uid="3f67ac66-fc82-43aa-8b2d-2a3c62b03920" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
2022-06-09T10:05:18.842343Z ERROR validation{host="policy-server-default-c9f594c98-cpn8x" policy_id="namespaced-system-upgrade-privileged-pods" kind="Pod" kind_group="" kind_version="v1" name="apply-os-upgrade-on-rancher-with-512d25c1140afd37158df753-cdgzd" namespace="system-upgrade" operation="CREATE" request_uid="02a2603b-a5b6-4a7b-a0ed-393d0bddf644" resource="pods" resource_group="" resource_version="v1" subresource=""}:policy_eval:validate{self=PolicyEvaluator { id: "namespaced-system-upgrade-privileged-pods", settings: {"modifyImagesWithDigest": Bool(true), "signatures": Array([Object({"annotations": Object({"env": String("prod")}), "image": String("quay.io/c3os/*"), "keyless": Array([Object({"issuer": String("https://token.actions.githubusercontent.com"), "subject": String("c3os-io")})])})])} }}: policy_evaluator::runtimes::wapc: callback evaluation failed policy_id=1 binding="kubewarden" operation="v1/verify" error="no signatures found for image: quay.io/c3os/c3os:opensuse-v1.21.4-31 "
```

## References
- https://docs.kubewarden.io/quick-start
- https://docs.kubewarden.io/distributing-policies/secure-supply-chain
- https://github.com/kubewarden/verify-image-signatures