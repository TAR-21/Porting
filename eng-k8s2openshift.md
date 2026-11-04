# Kubernetes to OpenShift Application Migration and Certification Task List

This document outlines the end-to-end tasks required to migrate an existing application running on Kubernetes (K8s) to Red Hat OpenShift, adopt the **Red Hat Universal Base Image (UBI)**, and obtain and maintain **Red Hat Container Certification** and **OpenShift Operator / Helm Certification**.

---

## 1. Overall Roadmap

| Phase | Primary Objective | Key Deliverables |
| --- | --- | --- |
| **Phase 1: Base Image Design & Migration** | Migrate to UBI, bundle licenses, implement multi-architecture support | UBI-compliant Containerfile / Dockerfile |
| **Phase 2: Security & Permissions Compliance** | Non-root execution (arbitrary UID support), SCC compliance, privilege drop | SCC-compliant, non-privileged container image |
| **Phase 3: Packaging** | Package as an Operator (OLM) or Helm Chart | ClusterServiceVersion (CSV) / Helm Chart |
| **Phase 4: Pre-validation & Testing** | Register project in Partner Connect, validate via Preflight CLI | Preflight validation reports & test logs |
| **Phase 5: Certification Submission & Publishing** | Submit to Pyxis, manage vulnerabilities, publish to Ecosystem Catalog | Certification badge, OperatorHub listing |

---

## 2. Phase-by-Phase Task Details

### Phase 1: UBI Migration & Build Design

As a mandatory prerequisite for Red Hat Container Certification, container images must be built on top of Red Hat Enterprise Linux (RHEL) or the Universal Base Image (UBI).

* [ ] **1.1 Select UBI Flavor**
* Choose the appropriate base image based on application requirements:
* `ubi9-micro`: No package manager included. Ideal for statically compiled binaries (Go, Rust, etc.).
* `ubi9-minimal`: Minimal package manager (`microdnf`) included. Suited for general lightweight microservices.
* `ubi9`: Full `dnf` stack included. Suitable when standard utilities and build tools are required.
* Official language runtime images: Python, Node.js, OpenJDK, .NET, etc.




* [ ] **1.2 Modernize Package Management & Build Instructions**
* Replace Debian (`apt`) or Alpine (`apk`) package manager commands with `dnf` or `microdnf`.
* Restrict package sources to official UBI repositories (`ubi-9-baseos`, `ubi-9-appstream`).
* Thoroughly clear caches to minimize image size (`microdnf clean all` / `rm -rf /var/cache/yum`).


* [ ] **1.3 Bundle License Files (Mandatory Requirement)**
* Place your software end-user license agreements (EULA) and third-party OSS licenses inside the `/licenses` directory within the container.


* [ ] **1.4 Multi-Architecture Build Support (Recommended / As Needed)**
* Build and publish multi-arch manifest lists using `podman build --platform=linux/amd64,linux/arm64` and push them to target registries (such as Quay.io).



---

### Phase 2: OpenShift Security & Runtime Compliance

Ensure workloads run seamlessly under OpenShift’s default Security Context Constraints (SCC: `restricted-v2`).

* [ ] **2.1 Non-Root & Arbitrary UID Execution Support**
* Avoid setting `USER 0` (root) in the Dockerfile; specify a non-privileged user (e.g., `USER 1001`).
* To support OpenShift's randomly allocated UIDs at runtime, assign root group ownership (`gid=0`) and group write permissions to necessary application directories:
```dockerfile
RUN chown -R 1001:0 /app && \
    chmod -R g=u /app
USER 1001

```




* [ ] **2.2 Eliminate Privileges & Linux Capabilities**
* Prohibit `securityContext.privileged: true`.
* Drop unnecessary Linux Capabilities (such as `NET_ADMIN`, `SYS_ADMIN`) and verify standard execution under restricted capabilities.


* [ ] **2.3 Remove SUID/SGID Binaries**
* Remove unnecessary binaries with SUID/SGID bits set during the build stage to mitigate security risks.


* [ ] **2.4 Align Ingress & Storage Configurations**
* Verify alignment between K8s Ingress and OpenShift `Route` (Edge / Passthrough / Re-encrypt) or standard Ingress resources.
* Remove cloud-provider-specific annotations and replace them with portable settings.
* Verify PersistentVolumeClaim (PVC) access modes (RWO/RWX) and file permissions (UID/GID compatibility).



---

### Phase 3: OpenShift Packaging (Operator or Helm)

To achieve OpenShift Certification, workloads must be packaged and distributed as an Operator (recommended) or a Helm Chart.

#### A. Operator Approach (Operator Lifecycle Manager / OLM)

* [ ] **3.1 Develop the Operator**
* Implement controllers using the Operator SDK (Go, Ansible, or Helm-based).


* [ ] **3.2 Configure ClusterServiceVersion (CSV)**
* Define Operator metadata, supported OpenShift versions, and required annotations.
* Pin all container images consumed by the Operator and managed workloads in the `relatedImages` section using content digests (`sha256:`).
* Provide branding assets (icon, descriptions, support and provider details).


* [ ] **3.3 Enforce Least-Privilege RBAC**
* Scope ServiceAccount permissions strictly to namespaces (Roles) and avoid widespread use of ClusterRoleBindings.



#### B. Helm Chart Approach

* [ ] **3.4 Adapt Helm Chart**
* Ensure compliance with Helm v3 standards and include templates compatible with OpenShift-native resources (e.g., Routes).



---

### Phase 4: Pre-validation & Testing via Red Hat Partner Connect

Verify compliance locally and against test clusters using the official Red Hat certification tool, `preflight` CLI.

* [ ] **4.1 Create Project in Partner Connect**
* Log in to [Red Hat Partner Connect](https://connect.redhat.com/) and register the organization account.
* Create certification projects (Container Project / Operator Project) and generate Pyxis API tokens.


* [ ] **4.2 Run Container Certification Test via Preflight CLI**
* Execute preflight against the target container image:
```bash
preflight check container <IMAGE_URI> \
  --pyxis-api-token=<TOKEN> \
  --certification-project-id=<PROJECT_ID> \
  --docker-config=~/.docker/config.json

```


* Ensure all checks pass (UBI base layer verification, layer structure, license presence, and vulnerability scans).


* [ ] **4.3 Run Operator Certification Test via Preflight CLI**
* Execute preflight against the Operator Bundle:
```bash
preflight check operator <BUNDLE_IMAGE_URI> \
  --pyxis-api-token=<TOKEN> \
  --certification-project-id=<PROJECT_ID> \
  --docker-config=~/.docker/config.json

```


* Verify that OLM specification compliance, CRD schema validity, and non-privileged execution tests pass.



---

### Phase 5: Certification Submission, Maintenance & Catalog Publishing

* [ ] **5.1 Submit Certification Results**
* Run Preflight with the `--submit` flag to upload test results directly to the Red Hat certification backend (Pyxis).


* [ ] **5.2 Maintain Container Health Index**
* Review vulnerability scanning results from Red Hat scanners and maintain a healthy score (Health Index A to C) with zero unresolved Critical or Important CVEs.


* [ ] **5.3 Red Hat Review & Catalog Publication**
* Complete the product listing profile and support metadata in the Partner Connect portal and submit for final approval.
* Once approved, publish the solution to the **Red Hat Ecosystem Catalog** and cluster-native **OperatorHub**.


* [ ] **5.4 Continuous Re-certification (CI/CD Pipeline Integration)**
* Integrate Preflight CLI verification and automated submission into CI/CD pipelines to ensure continuous certification across application releases and OpenShift platform updates.
