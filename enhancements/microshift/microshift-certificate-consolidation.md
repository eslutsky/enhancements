---
title: microshift-certificate-consolidation
authors:
  - "@eslutsky"
reviewers:
  - TBD
approvers:
  - TBD
api-approvers:
  - "None"
creation-date: 2026-07-29
last-updated: 2026-09-10
tracking-link:
  - https://redhat.atlassian.net/browse/OCPSTRAT-2900
see-also:
  - "/enhancements/microshift/microshift-apiserver-certs.md"
  - "/enhancements/microshift/microshift-certificate-rotation.md"
replaces:
  - N/A
superseded-by:
  - N/A
---

# MicroShift Certificate Authority Consolidation

## Summary

MicroShift currently operates 12 Certificate Authorities (CAs) and 18 leaf
certificates inherited from OpenShift's multi-node architecture. On a
single-node edge device this complexity provides no additional security
benefit while increasing operational overhead and complicating future work
such as controlled certificate renewal. This enhancement consolidates the
PKI to 5 CAs (2 new, 3 unchanged) and reduces the KAS serving certificates
from 3 to 1 using Subject Alternative Names (SANs), aligning with upstream
Kubernetes best practices and ProdSec recommendations for the single-node
edge deployment model.

## Motivation

A ProdSec review conducted in March 2026 identified that MicroShift's 12 CAs
serve no security purpose on a single-node device. In OpenShift, separate CAs
enable independent trust domains across nodes — for example, one node's kubelet
does not need to trust another node's API server client certificate. On a
single-node device all certificates are consumed by processes on the same host,
making trust domain separation meaningless.

The current structure also creates maintenance burden: each CA has its own
validity lifecycle, its own trust bundle membership, and its own renewal
logic. Consolidation simplifies certificate renewal (OCPSTRAT-2899), reduces
the number of files on disk, and makes the PKI easier to reason about for
both developers and field engineers.

### User Stories

* As a MicroShift platform engineer, I want fewer CAs to manage so that
  certificate renewal and troubleshooting are simpler.
* As a MicroShift operator, I want the certificate layout to reflect the
  single-node deployment model so that I can understand and verify the PKI
  without deep OpenShift knowledge.
* As a MicroShift developer, I want a simpler `certSetup()` function so that
  future changes (controlled cert renewal, multi-node) are easier to implement.
* As a security reviewer, I want the PKI to follow upstream Kubernetes
  recommendations (client CA, serving CA, etcd CA) so that audits are
  straightforward.

### Goals

* Reduce the number of managed CAs from 12 to 5.
* Consolidate the 3 KAS serving certificates into 1 SAN-based certificate.
* Preserve all leaf certificate identities (CN/O fields) so that Kubernetes
  RBAC authorization is unaffected.
* Provide an automatic upgrade migration path requiring no operator intervention.
* Support OS-level rollback via greenboot/ostree.

### Non-Goals

* Changing the etcd-signer, service-ca, or ingress-ca in any way.
* Implementing the `microshift certs` CLI subcommands, PKI inventory abstraction,
  or configurable certificate validity — those are covered by OCPSTRAT-2899,
  which co-ships with this enhancement.
* Changing leaf certificate validity periods or renewal thresholds.
* Modifying the `certchains` builder framework API itself.
* Changing the service account signing key mechanism.

## Proposal

### Current State

MicroShift maintains 12 CAs organized into three categories:

**Client certificate signers (5 CAs):**

| CA | Leaf certificates signed |
|----|--------------------------|
| kube-control-plane-signer | kube-controller-manager, kube-scheduler, cluster-policy-controller, route-controller-manager |
| kube-apiserver-to-kubelet-signer | kube-apiserver-to-kubelet-client, metrics-server-kubelet-client |
| admin-kubeconfig-signer | admin-kubeconfig-client, openshift-observability-client |
| kubelet-csr-signer-signer → kube-csr-signer (sub-CA) | kubelet-client, kubelet-server |
| aggregator-signer | aggregator-client |

**Serving certificate signers (3 CAs):**

| CA | Leaf certificates signed |
|----|--------------------------|
| kube-apiserver-external-signer | kube-external-serving |
| kube-apiserver-localhost-signer | kube-apiserver-localhost-serving |
| kube-apiserver-service-network-signer | kube-apiserver-service-network-serving |

**Unchanged signers (4 CAs):**

| CA | Leaf certificates signed | Reason unchanged |
|----|--------------------------|------------------|
| etcd-signer | apiserver-etcd-client, etcd-peer, etcd-serving | Isolated trust domain |
| service-ca | route-controller-manager-serving | Independent lifecycle |
| ingress-ca | router-default-serving | Independent lifecycle |

Note: kubelet-csr-signer-signer contains a sub-CA (kube-csr-signer) making it
effectively 2 CAs, for a total of 12 (11 root + 1 sub-CA).

### Target State

**New CAs:**

`client-ca` replaces 5 CAs (kube-control-plane-signer,
kube-apiserver-to-kubelet-signer, admin-kubeconfig-signer,
kubelet-csr-signer-signer/kube-csr-signer, aggregator-signer) and signs all
client certificates:

| Leaf certificate | CN | O (groups) |
|------------------|----|------------|
| kube-controller-manager | system:kube-controller-manager | — |
| kube-scheduler | system:kube-scheduler | — |
| cluster-policy-controller | system:kube-controller-manager | — |
| route-controller-manager | system:serviceaccount:openshift-route-controller-manager:route-controller-manager-sa | — |
| kube-apiserver-to-kubelet-client | system:kube-apiserver | kube-master |
| metrics-server-kubelet-client | system:metrics-server | — |
| admin-kubeconfig-client | system:admin | system:masters |
| openshift-observability-client | openshift-observability-client | — |
| kubelet-client | system:node:\<nodename\> | system:nodes |
| aggregator-client | system:openshift-aggregator | — |

`serving-ca` replaces 3 CAs (kube-apiserver-external-signer,
kube-apiserver-localhost-signer, kube-apiserver-service-network-signer) and
signs serving certificates:

| Leaf certificate | SANs |
|------------------|------|
| kube-apiserver-serving | kubernetes, kubernetes.default, kubernetes.default.svc, kubernetes.default.svc.cluster.local, openshift, openshift.default, openshift.default.svc, openshift.default.svc.cluster.local, api.\<basedomain\>, api-int.\<basedomain\>, \<advertise-address\>, \<service-ip\>, localhost, 127.0.0.1, ::1, \<hostname\>, \<node-ip\>, \<subject-alt-names\> |
| kubelet-server | \<hostname\>, \<node-ip\> |

**Unchanged CAs:** etcd-signer, service-ca, ingress-ca remain identical.

### Certificate Identity Preservation

All leaf certificate Common Name (CN) and Organization (O) fields remain
identical to their current values. Kubernetes RBAC determines identity from
these fields, not from which CA signed the certificate. Changing the parent
CA has zero impact on authorization.

### KAS Serving Certificate Consolidation

Currently, the KAS uses 3 separate serving certificates for external,
localhost, and service-network access, each signed by a different CA. The
KAS `dynamiccertificates` package selects which certificate to serve based
on SNI or destination IP matching.

With consolidation, a single serving certificate contains all SANs from the
3 old certificates. Since all connections to the KAS will match this single
certificate regardless of the destination IP or hostname used, the
`dynamiccertificates` selection logic continues to work — it simply always
matches the same certificate.

### Workflow Description

**Fresh install:**
1. MicroShift starts and calls `initCerts()`.
2. `certSetup()` builds the new 5-CA hierarchy.
3. All leaf certs, kubeconfigs, and trust bundles are generated.
4. MicroShift starts normally.

**Upgrade from pre-consolidation version:**
1. New MicroShift binary starts and calls `initCerts()`.
2. Migration logic detects the old CA directory layout by checking for the
   existence of `<datadir>/certs/kube-control-plane-signer/`.
3. The entire `certs/` directory is renamed to
   `certs.backup.<version>/`.
4. `certSetup()` finds no certs directory, generates everything fresh with
   the new hierarchy.
5. Kubeconfigs are regenerated with the new serving-ca as the trust anchor.
6. All pods restart and receive new service account tokens.

Note: upgrade causes a brief service disruption — all certificates are
regenerated and all pods are restarted. External clients that have cached or
distributed the old CA certificate must update their trust configuration to
the new CA material (available from the regenerated admin kubeconfig or the
updated CA bundle files).

**Rollback (greenboot/ostree):**
1. If greenboot detects an unhealthy system after upgrade, it triggers an
   atomic rollback to the previous OS commit (via ostree).
2. The old MicroShift binary starts, finds no `certs/` directory (the backup
   has a different name), and regenerates all certs from scratch using the
   old hierarchy with new key material.
3. MicroShift starts normally with the old CA layout.

Note: rollback atomicity applies to the OS commit only. Certificate
material is not restored from backup — new keys are generated for the old
layout. External clients that updated their trust configuration for the new
CA must update again to the freshly generated old-layout CA.

**Backup cleanup:**
The `certs.backup.<version>/` directory is left on disk and not
automatically deleted. Documentation will advise operators to remove it after
confirming the upgrade is stable. Edge devices have sufficient disk capacity
for one backup.

### CA Bundle Changes

| Bundle file | Current contents | New contents |
|-------------|-----------------|--------------|
| `ca-bundle/client-ca.crt` | 5 CA certs concatenated | single client-ca cert |
| `ca-bundle/kubelet-ca.crt` | kubelet-csr-signer CA | client-ca cert |
| `ca-bundle/kubelet-serving-ca.crt` | kubelet-csr-signer CA | serving-ca cert |
| `ca-bundle/service-account-token-ca.crt` | 3 KAS serving CAs | serving-ca cert |
| `ca-bundle/ca-bundle.crt` | all serving CAs | serving-ca + etcd-signer certs |

ConfigMaps and Secrets exposed to Kubernetes maintain the same names and
namespaces. Consumer-side configuration does not change.

### API Extensions

N/A — this enhancement does not modify any Kubernetes API resources, CRDs,
webhooks, or aggregated API servers.

### Topology Considerations

#### Hypershift / Hosted Control Planes

N/A — MicroShift does not participate in Hypershift topologies.

#### Standalone Clusters

N/A — this enhancement is specific to MicroShift.

#### Single-node Deployments or MicroShift

This enhancement is designed specifically for MicroShift's single-node edge
deployment model. The consolidation is safe precisely because all certificate
consumers run on the same node. The change reduces resource consumption
slightly (fewer files on disk, fewer CA key pairs to generate at startup).

#### OpenShift Kubernetes Engine (OKE)

N/A — MicroShift does not depend on features excluded from OKE.

### Implementation Details/Notes/Constraints

**Files modified:**

| File | Change |
|------|--------|
| `pkg/util/cryptomaterial/certinfo.go` | New CA path functions (`ClientCADir`, `ServingCADir`), new leaf cert dir functions, remove 9 old CA path functions |
| `pkg/cmd/init.go` | Rebuild `certSetup()` with 2 new CAs, add `migrateCertsIfNeeded()`, update `initKubeconfigs()` GetCertKey paths |
| `pkg/components/certificateauthority.go` | Update `CertificateAuthorityResources` from 12 entries to 5 |
| `pkg/controllers/kube-apiserver.go` | Single serving cert, updated CA paths for aggregator and kubelet |
| `pkg/controllers/kube-controller-manager.go` | Updated serving cert and CSR signer references |
| `pkg/controllers/kube-scheduler.go` | Updated serving cert and healthcheck CA |
| `pkg/node/kubelet.go` | Updated serving cert dir |
| `pkg/components/metrics.go` | Updated admin CA reference to client-ca |
| `pkg/telemetry/gather.go` | Updated client cert dir (automatic via certinfo.go change) |

### PKI Inventory Integration

OCPSTRAT-2899 introduces a PKI inventory that catalogs all managed certificates
by role (CA, serving, client, peer) and tracks parent-child signing relationships.
This enhancement updates that inventory from the current 12-CA layout to the new
5-CA layout:

| Role | Old entries | New entries |
|------|-------------|-------------|
| Rotatable CAs | kube-control-plane-signer, kube-apiserver-to-kubelet-signer, admin-kubeconfig-signer, kubelet-csr-signer-signer, kube-csr-signer (sub-CA), kube-apiserver-external-signer, kube-apiserver-localhost-signer, kube-apiserver-service-network-signer | client-ca, serving-ca |
| Fixed CAs | etcd-signer, service-ca, ingress-ca | etcd-signer, service-ca, ingress-ca (unchanged) |
| Serving certs | kube-external-serving, kube-apiserver-localhost-serving, kube-apiserver-service-network-serving | kube-apiserver-serving (single SAN-based cert) |

The inventory abstraction ensures that `microshift certs renew --ca` cascades
renewal to all descendant certificates correctly under the new hierarchy without
requiring changes to the CLI code.

**Service account tokens:** Service account JWTs are signed and verified
using key files (`service-account-key-file`), not by a CA. The signing key
is not changed by this enhancement (see Non-Goals). Existing tokens remain
cryptographically valid. Pods are restarted as part of the migration (new
cert material), and they receive fresh projected tokens automatically via
the standard Kubernetes token volume mechanism. No user action required.

**Certificate validity:** The new CAs (`client-ca`, `serving-ca`) use
`LongLivedCertificateValidity` (10 years). This is a deliberate increase
from the 1-year validity of the CAs they replace: a consolidated root CA
on an embedded single-node device has no practical benefit from frequent
rotation, and a longer CA lifetime reduces operational burden. Leaf
certificate validity and renewal thresholds are unchanged (see Non-Goals).

**certchains framework:** No changes to the builder framework itself. The
same `NewCertificateSigner`, `WithClientCertificates`,
`WithServingCertificates`, `WithCABundle`, and `Complete` APIs are used.

### Risks and Mitigations

* **Risk: External systems caching old CA certificates.**
  Mitigation: MicroShift edge devices are typically not integrated with
  external CA trust stores. The admin kubeconfig is regenerated on upgrade.
  If operators have distributed the old CA cert to external systems, they
  must update those systems after upgrade.

* **Risk: Service account token invalidation during upgrade.**
  Mitigation: MicroShift restarts all workloads on upgrade. Pods receive
  new tokens automatically. Operators using long-lived extracted tokens
  (anti-pattern) must re-extract after upgrade.

* **Risk: Migration failure leaves no certs directory.**
  Mitigation: The `os.Rename` operation is atomic on the same filesystem.
  If it fails, the old `certs/` directory remains intact and MicroShift
  continues with the old layout. If it succeeds but `certSetup()` fails,
  greenboot triggers a rollback.

### Drawbacks

* **Divergence from OpenShift:** The consolidated CA layout differs from
  OCP's multi-CA structure. This is intentional — the ProdSec review
  concluded that the multi-CA structure serves no purpose on a single node.
  Future multi-node MicroShift deployments may need to re-introduce
  separate trust domains, but that would be a separate enhancement.

* **One-time upgrade disruption:** All certificates are regenerated, which
  means a brief period during startup where services are restarting with
  new certs. On a single-node device with MicroShift managing all
  components, this is functionally equivalent to a fresh start.

## Design Details

### Open Questions [optional]

None.

### Test Plan

**Unit tests:**
* Verify the new chain builder produces the correct 5-CA hierarchy.
* Verify the consolidated KAS serving cert contains all expected SANs.
* Verify leaf cert CN/O fields match their original values.
* Verify migration detection correctly identifies old vs. new layouts.
* Verify migration creates a backup directory and removes the old certs dir.

**Integration tests (Robot Framework):**
* **Fresh install test:** Install new MicroShift version, verify cert
  directory structure matches the new layout, verify services are healthy.
* **Migration test:** Deploy old version, upgrade to new version, verify
  backup directory created, new CAs exist, old CAs absent, services healthy,
  `oc login` works.
* **Rollback test:** Upgrade, trigger greenboot rollback, verify old version
  regenerates certs and runs normally.
* **Cert renewal test:** Advance system clock past expiry, restart, verify
  certs renewed under new CAs.

**Manual verification checklist:**
* `oc get csr` — no pending CSRs stuck.
* `openssl verify` — leaf certs chain to correct CA.
* `oc login` — admin kubeconfig works.
* Workloads (pods, routes) functional after upgrade.
* Service account tokens valid (API calls from pods succeed).

### Graduation Criteria

#### Dev Preview -> Tech Preview

N/A — this is an internal infrastructure change, not a user-facing feature.
It ships as GA in 5.1.

#### Tech Preview -> GA

* All unit and integration tests passing.
* Upgrade testing from 5.0 to 5.1 validated.
* Rollback testing validated with greenboot.
* ProdSec sign-off on the new layout.

#### Removing a deprecated feature

The old CA layout is removed in the same release. No deprecation period is
needed because:
1. The CA directories are internal implementation details, not a public API.
2. The backup-and-regenerate migration handles the transition automatically.
3. Kubernetes Secret names and namespaces are preserved.

### Upgrade / Downgrade Strategy

**Upgrade (5.0 → 5.1):**
1. New binary detects old cert layout on first startup.
2. Old `certs/` directory is atomically renamed to `certs.backup.<version>/`.
3. Fresh cert generation creates the new layout.
4. All services restart with new certificates.

**Downgrade (5.1 → 5.0 via greenboot rollback):**
1. Old binary starts, finds no `certs/` directory.
2. Existing cert regeneration logic creates the old layout from scratch.
3. Services start normally.

No manual intervention is required in either direction.

### Version Skew Strategy

N/A — MicroShift runs all components at the same version on a single node.
There is no version skew between control plane and kubelet.

### Operational Aspects of API Extensions

N/A — no API extensions are introduced.

#### Failure Modes

* **Migration rename fails (permissions, disk full):** `initCerts()` returns
  an error, MicroShift does not start. The old `certs/` directory is
  unchanged. Operator fixes the disk issue and restarts.

* **Fresh cert generation fails after migration:** `certs/` directory does
  not exist (was renamed), `certSetup()` fails. Greenboot detects unhealthy
  state and triggers rollback. Old binary regenerates certs from scratch.

#### Support Procedures

* **Detecting migration occurred:** Check for `certs.backup.*` directory
  under the data directory.
* **Verifying new layout:** `ls /var/lib/microshift/certs/` should show
  `client-ca/`, `serving-ca/`, `etcd-signer/`, `service-ca/`, `ingress-ca/`,
  and `ca-bundle/`.
* **Reverting manually:** Stop MicroShift, remove `certs/`, rename
  `certs.backup.<version>/` back to `certs/`, restart. Old certs will be
  used until the next upgrade triggers migration again.

## Implementation History

* 2026-07-29: Initial enhancement proposal.

## Alternatives (Not Implemented)

* **Bridge trust period (keep old CAs in bundles for one release cycle).**
  This would allow old and new CAs to be trusted simultaneously during a
  transition period, with old CAs pruned in 5.2. Rejected because on a
  single-node device there are no external peers that need to gradually
  transition trust. The backup-and-regenerate approach is simpler and
  achieves the same result in one step.

* **In-place re-signing (keep old CA directories, just change which CA
  signs each leaf cert).** This would preserve the directory structure
  while changing the signing relationships. Rejected because it does not
  achieve the goal of simplifying the on-disk layout and leaves dead CA
  key material on disk.


## Infrastructure Needed [optional]

N/A
