# Troubleshooting Guide

This guide covers common issues users might run into when using Cluster API Provider Linode.
This list is work-in-progress, please feel free to open a PR to add this guide if you find
that useful information is missing.

## Examples of common issues

### No Linode resources are getting created

This could be due to the `LINODE_TOKEN` either not being set in your environment or expired.
If expired, [provision a new token](../topics/getting-started.md#prerequisites) and optionally
set the "Expiry" to "Never" (default expiry is 6 months).

### Instance creation is blocked by capacity or an account limit

CAPL classifies exactly two Linode instance-create failures:

| Condition reason | Cause | Recovery |
| --- | --- | --- |
| `InsufficientCapacity` | The Linode API returns HTTP 403 indicating that the requested plan is not currently available in the selected region. | Wait for capacity, choose another plan or region for replacement Machines, or contact Support. |
| `AccountLimitReached` | The Linode API returns HTTP 400 indicating that the account's active-services limit has been reached. | Remove unused services or contact Support to request a limit increase. |

Both `PreflightCreated` and `Ready` on the `LinodeMachine` become `False`, with
the classified reason and original error message for diagnostics. CAPI mirrors
`Ready` into the owning Machine's `InfrastructureReady` condition. Inspect them
on the management cluster:

```bash
kubectl get linodemachine "$LINODE_MACHINE_NAME" -n "$NAMESPACE" \
  -o jsonpath='{range .status.conditions[?(@.type=="PreflightCreated")]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}{range .status.conditions[?(@.type=="Ready")]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'
kubectl get machine "$MACHINE_NAME" -n "$NAMESPACE" \
  -o jsonpath='{range .status.conditions[?(@.type=="InfrastructureReady")]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'
```

The classified conditions survive HTTP 408, 429, all 5xx responses, SDK
pseudo-errors, and non-API errors such as transport failures. A later recognized
failure replaces the reason. A different API 4xx response (other than 408 or 429)
retires it: `PreflightCreated` resumes its existing `CreateError` diagnostic and
a classified `Ready` condition is removed. The existing HTTP 400 handler can
subsequently set `Ready=False/CreateError`.

Successful creation or adoption of an existing instance sets `PreflightCreated=True`
and removes only a classified `Ready` condition. No neutral reason replaces it:
while the `LinodeMachine` has no `Ready` condition, CAPI reports the Machine's
`InfrastructureReady` with its standard fallback reason (`NotReady`). Readiness
and provisioning completion still follow the normal lifecycle.

Retry behavior is unchanged. Before the five-minute preflight timeout, supported
transient errors use their existing fixed-delay retries; other create failures
already trigger error backoff through the caller's `nil ProviderID` error.
After the timeout, failed attempts return the original create error. The timeout
counts from the first failed create, including an earlier generic error;
False-to-False reason changes retain the persisted first-failure timestamp.
Error retries can back off to about 17 minutes, so a classified reason may remain
after the underlying issue is resolved until another attempt succeeds.

Existing generic diagnostics, events, and legacy failure fields remain unchanged
outside preservation or retirement of this signal. `CreateError` is an internal
diagnostic, not another classified category; absence of a classified reason does
not rule out other provisioning problems.

### One or more control plane replicas are missing

Take a look at the `KubeadmControlPlane` controller logs and look for any potential errors:

```bash
kubectl logs deploy/capi-kubeadm-control-plane-controller-manager -n capi-kubeadm-control-plane-system manager
```

In addition, make sure all pods on the workload cluster are healthy, including pods in the `kube-system` namespace.

Otherwise, [ensure that the linode-ccm is installed on your workload cluster via CAAPH](../topics/addons.md#ccm).

### Nodes are in NotReady state

Make sure [a CNI is installed on the workload cluster](../topics/addons.md#cni)
and that all the pods on the workload cluster are in running state. 

If the Cluster is labeled with `cni: <cluster-name>-cilium`, check that the \<cluster-name\>-cilium `HelmChartProxy` is installed in
the management cluster and that the `HelmChartProxy` is in a `Ready` state:

```bash
kubectl get cluster $CLUSTER_NAME --show-labels
```

```bash
kubectl get helmchartproxies
```

## Checking CAPI and CAPL resources

To check the progression of all CAPI and CAPL resources on the management cluster you can run:

```bash
kubectl get cluster-api
```

## Looking at the CAPL controller logs

To check the CAPL controller logs on the management cluster, run:

```bash
kubectl logs deploy/capl-controller-manager -n capl-system manager
```

### Checking cloud-init logs (Debian / Ubuntu)

[Cloud-init](https://www.linode.com/docs/guides/applications/configuration-management/cloud-init/)
logs can provide more information on any issues that happened when running the bootstrap script.

```admonish warning
Not all Debian and Ubuntu images available from Linode support cloud-init! Please see the
[Availability section of the Linode Metadata Service Guide](https://www.linode.com/docs/products/compute/compute-instances/guides/metadata/#availability).

You can also see which images have cloud-init support via the [linode-cli](https://www.linode.com/docs/products/tools/cli/get-started/):

`linode-cli images list | grep cloud-init`

```

Please refer to the [Troubleshoot Metadata and Cloud-Init section of the Linode Metadata Service Guide](https://www.linode.com/docs/products/compute/compute-instances/guides/metadata/?tabs=linode-api%2Cmacos#troubleshoot-metadata-and-cloud-init).

## Increasing Linode API timeout values
If the Linode API is slow to provision resources and you need to increase the timeout for API calls, you can set the LINODE_CLIENT_TIMEOUT environment variable to a higher value (in seconds). CAPL will automatically use this value when interacting with the Linode API.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: capl-controller-manager
spec:
  template:
    spec:
      containers:
        - name: manager
          env:
            - name: LINODE_CLIENT_TIMEOUT
              value: "60"  # Timeout in seconds
```
