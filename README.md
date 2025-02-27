# Kubewarden policy verify-image-signatures

## Description

This policy validates Sigstore signatures for containers, init container and ephemeral container that match the name provided
in the `image` settings field. It will reject the Pod if any validation fails.
If all signature validation pass or there is no container that matches the image name, the Pod will be accepted.

This policy also mutates matching images to add the image digest, therefore the version of the deployed image can't change. 
This mutation can be disabled by setting `modifyImagesWithDigest` to `false`.

It can also reject all workload resources that contain containers: Deployments, ReplicaSets, StatefulSets, DaemonSets, Jobs, CronJobs, ReplicationControllers
For this you need to add these resources in the rules field of the policy. 

Example of a policy that will reject any workload resources mentioned before:

``` yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: verify-image-signatures
spec:
  module: ghcr.io/kubewarden/policies/verify-image-signatures:v0.1.7
  rules:
  - apiGroups: ["", "apps", "batch"]
    apiVersions: ["v1"]
    resources: ["pods", "deployments", "statefulsets", "replicationcontrollers", "jobs", "cronjobs"]
    operations:
    - CREATE
    - UPDATE
  mutating: true
  settings:
    signatures:
    - image: "*"
      githubActions:
        owner: "kubewarden"
        repo: "app-example" 
```

See the [Secure Supply Chain docs in Kubewarden](https://docs.kubewarden.io/distributing-policies/secure-supply-chain) for more info.

## Settings

The policy takes a list of signatures. A signature can be of four types: GitHub actions, public key, keyless exact match or keyless prefix. Each signature
has an `image` field which will be used to select the matching containers in the pod that will be evaluated.
`image` supports wildcard. For example, `ghcr.io/kubewarden/*` will match all images from the kubewarden ghcr repo.

Signature types:

1. GitHub actions. It will verify that all images were signed for a GitHub action with the `kubewarden` owner and in the repo `app-example`.

  ``` yaml
  signatures:
  - image: "ghcr.io/kubewarden/*"
    githubActions:
      owner: "kubewarden"
      repo: "app-example" #optional
  ```

2. Keyless subject prefix. It will verify that the issuer is `https://token.actions.githubusercontent.com` and the subject starts with `https://github.com/kubewarden/app-example/.github/workflows/ci.yml@refs/tags/`
   `urlPrefix` is sanitized to prevent typosquatting.

  ``` yaml
  signatures:
  - image: "ghcr.io/kubewarden/*"
    keylessPrefix:
      - issuer: "https://token.actions.githubusercontent.com"
        urlPrefix: "https://github.com/kubewarden/app-example/.github/workflows/ci.yml@refs/tags/"
  ``` 


3. Keyless exact match. It will verify that the issuer is `https://token.actions.githubusercontent.com` and the subject is `kubewarden`. It will not modify the image with the digest.

  ``` yaml
  modifyImagesWithDigest: false #optional. default is true
  signatures:
    - image: "ghcr.io/kubewarden/*"
      keyless:
        - issuer: "https://token.actions.githubusercontent.com"
          subject: "kubewarden"
  ``` 

4. Public key. It will verify that all images were signed with the two public keys provided and contains the `env: prod` annotation.

  ``` yaml
  signatures:
    - image: "ghcr.io/kubewarden/*"
      pubKeys: 
        - "-----BEGIN PUBLIC KEY-----xxxxx-----END PUBLIC KEY-----"
        - "-----BEGIN PUBLIC KEY-----xxxxx-----END PUBLIC KEY-----"
      annotations: #optional
        env: prod
  ``` 

5. Certificate. It will verify that the image has been signed using all the
  certificates provided by the user.
  The certificates must be PEM encoded. Optionally the settings can have
  the list of PEM encoded certificates that can create the `certificateChain`
  used to verify the given `certificate`.
  The `requireRekorBundle` should be set to `true` to have a stronger
  verification process. When set to `true`, the signature must have a Rekor
  bundle and the signature must have been created during the validity
  time frame of the `certificate`.

  The following configuration requires all the container images coming from
  `registry.acme.org/secure-project` to be signed both by Alice and by Bob.

  ```yaml
  signatures:
    - image: "registry.acme.org/secure-project/*"
      certificates:
      - |
        -----BEGIN CERTIFICATE-----
        alice's cert
        -----END CERTIFICATE-----
      - |
        -----BEGIN CERTIFICATE-----
        bob's cert
        -----END CERTIFICATE-----
      certificateChain:
      - |
        -----BEGIN CERTIFICATE-----
        <intermediate cert>
        -----END CERTIFICATE-----
      - |
        -----BEGIN CERTIFICATE-----
        <root CA>
        -----END CERTIFICATE-----
      requireRekorBundle: true
      annotations: #optional
        env: prod
  ```
