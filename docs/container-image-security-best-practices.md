# Container Image Security & Integrity Program

## Executive Summary
This document outlines a formalized approach for safeguarding container images throughout our delivery pipeline. The goal is to ensure that every artifact deployed to customer environments is authenticated, vulnerability assessed, and monitored for tampering. The guidance below reflects current industry best practices and maps directly to our delivery flow: `Dev Repo → Jenkins → DockerHub / ECR → K8s / Lambda → Customer Environments`.

## 1. Foundational Controls: Image Signing and Integrity

### 1.1 Key Management and Signing Workflow
1. Generate a dedicated signing key pair for the build system using Cosign.
2. Store the private key (`cosign.key`) in a managed secret store such as AWS Secrets Manager or HashiCorp Vault; distribute the public key (`cosign.pub`) to verification services.
3. Integrate Cosign signing into the CI/CD pipeline immediately after the image is pushed to the registry.

```bash
# Generate signing material
cosign generate-key-pair

# Sign images during the build pipeline
cosign sign --key cosign.key <account-id>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>
```

### 1.2 Integrity Enforcement Prior to Deployment
* **Kubernetes (EKS):** Deploy the Sigstore Policy Controller and configure a `ClusterImagePolicy` that references the Cosign public key. Admission requests for unsigned images are rejected automatically.
* **AWS Lambda:** Perform Cosign verification as a mandatory pre-deployment step. Only Lambda updates that pass signature validation progress to `aws lambda update-function-code`.

```bash
cosign verify --key cosign.pub <account-id>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>
```

### 1.3 Cryptographic Digests and Layer Verification
* Capture SHA-256 digests for reference builds and verify them before deployment or during incident response.
* Preserve a manifest of known-good layers (via `docker inspect`) and compare it during integrity checks.

```bash
docker save <image>:<tag> | sha256sum > image.sha256
docker save <image>:<tag> | sha256sum -c image.sha256
```

## 2. Vulnerability Management

### 2.1 Continuous Image Scanning
Integrate scanners such as Trivy or Prisma Cloud Compute (Twistlock) into the CI pipeline. Configure the pipeline to fail on high or critical findings unless explicitly waived through a governance process.

```bash
trivy image --exit-code 1 --severity HIGH,CRITICAL <image>:<tag>
```

### 2.2 Registry Hygiene
* Enforce private registry access with transport security (e.g., ECR policies denying insecure connections).
* Enable immutable tags and lifecycle policies to limit retained images and reduce attack surface.

## 3. Deployment-Time Protections

### 3.1 Kubernetes Admission Controls
1. Install the Sigstore Policy Controller release manifest.
2. Apply a `ClusterImagePolicy` that limits allowable registries and enforces Cosign signatures.
3. Combine with RBAC and namespace isolation to control access to image pull secrets.

```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
    - glob: "<account-id>.dkr.ecr.*.amazonaws.com/*"
  authorities:
    - key:
        publicKeys:
          - data: |
              -----BEGIN PUBLIC KEY-----
              <cosign.pub contents>
              -----END PUBLIC KEY-----
```

### 3.2 AWS Lambda Release Gate
* Add a verification stage in Jenkins (or equivalent CI) that invokes `cosign verify` against the candidate Lambda image.
* Abort the deployment if signature validation fails; otherwise call `aws lambda update-function-code`.

## 4. Runtime Controls and Monitoring

### 4.1 Hardened Workload Configuration
* Pin deployments to image digests (`image:tag@sha256:<digest>`) to prevent drift.
* Apply restrictive security contexts: non-root execution, read-only root filesystems, and explicit user IDs.

```yaml
containers:
  - name: secure-app
    image: <registry>/<image>:<tag>@sha256:<digest>
    imagePullPolicy: Always
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      readOnlyRootFilesystem: true
```

### 4.2 Observability and Alerting
* Emit verification results and runtime checks into Prometheus metrics (e.g., `image_verification_failures_total`).
* Configure alerting rules that escalate any verification failure as a critical incident.

```yaml
groups:
  - name: image-integrity
    rules:
      - alert: ImageTamperingDetected
        expr: image_verification_failures_total > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Image tampering detected"
          description: "Image verification failed for {{ $labels.image }}"
```

## 5. Governance, Compliance, and Customer Delivery

### 5.1 Audit Logging
* Enable Kubernetes audit policies focused on container creation events in customer namespaces.
* Retain logs for forensic analysis and compliance evidence.

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["pods"]
      - group: "apps"
        resources: ["deployments"]
    namespaces: ["customer-*"]
```

### 5.2 Contractual Controls
* Embed image digests or signature fingerprints within customer SLAs or license agreements.
* Provide a runtime integrity check that reports the deployed digest, enabling tamper detection even in customer-operated environments.

## 6. Implementation Roadmap

| Phase | Timeline | Focus |
| --- | --- | --- |
| Phase 1 – Foundation | Weeks 1–2 | Establish signing, scanning, and lifecycle policies. |
| Phase 2 – Verification | Weeks 3–4 | Enforce signature verification in Kubernetes and Lambda pipelines. |
| Phase 3 – Monitoring | Weeks 5–6 | Deploy runtime monitoring, alerting, and audit logging. |
| Phase 4 – Customer Safeguards | Weeks 7–8 | Harden customer environments with RBAC, image pull secrets, and namespace isolation. |

## 7. Metrics for Continuous Improvement
* **Vulnerability posture:** count of unresolved high/critical findings per image.
* **Verification success rate:** percentage of deployments passing signature verification gates.
* **Deployment cadence:** measurement of security control impact on delivery speed.
* **Compliance coverage:** adherence to required policies across teams.
* **Incident response time:** duration from verification failure to remediation.

## 8. Jenkins Reference Pipeline

```groovy
pipeline {
  agent any
  stages {
    stage('Build') {
      steps { sh 'docker build -t ${ECR_REPO}:${BUILD_NUMBER} .' }
    }
    stage('Scan') {
      steps { sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL ${ECR_REPO}:${BUILD_NUMBER}' }
    }
    stage('Sign') {
      steps { sh 'cosign sign --key cosign.key ${ECR_REPO}:${BUILD_NUMBER}' }
    }
    stage('Push') {
      steps {
        sh 'docker push ${ECR_REPO}:${BUILD_NUMBER}'
        sh 'docker tag ${ECR_REPO}:${BUILD_NUMBER} ${ECR_REPO}:latest'
        sh 'docker push ${ECR_REPO}:latest'
      }
    }
    stage('Verify') {
      steps { sh 'cosign verify --key cosign.pub ${ECR_REPO}:${BUILD_NUMBER}' }
    }
  }
  post {
    always { sh 'docker rmi ${ECR_REPO}:${BUILD_NUMBER} || true' }
  }
}
```

## 9. Enforceability Considerations

| Objective | Mechanism | Enforceability |
| --- | --- | --- |
| Authenticate delivered artifacts | Cosign signing backed by KMS-stored keys | ✅ Fully enforceable |
| Block unsigned workloads | Admission controllers (Sigstore Policy Controller, Kyverno, OPA) | ✅ Enforceable when enabled |
| Detect post-delivery tampering | Digest/signature verification and runtime attestations | ✅ Detectable |
| Prevent customer-side modifications | Not technically feasible (customer holds root control) | 🚫 Out of scope |
| Maintain contractual integrity | Include digests/signatures in SLAs and license terms | ⚙️ Organizational control |

## 10. Next Steps
1. Approve the adoption of Cosign for all container image builds.
2. Stand up the Sigstore Policy Controller in staging and validate enforcement behavior.
3. Integrate vulnerability scanning and signature verification gates into Jenkins pipelines.
4. Define ownership for monitoring dashboards and alert response procedures.
5. Socialize SLA updates with Customer Success to incorporate digest-based assurances.

---
By implementing these controls sequentially, the organization can materially reduce the risk of tampered or vulnerable container images reaching customer environments while maintaining auditable assurance throughout the delivery lifecycle.
