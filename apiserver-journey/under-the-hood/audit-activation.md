# From the audit flags to WithAudit

`WithAudit` receives a `sink` and a `policy`, and returns the next handler unchanged when either is missing. This note follows both values back to the kube-apiserver flags they come from. Source: **Kubernetes v1.37.2** (commit [`c50e5395f1ae`](https://github.com/kubernetes/kubernetes/tree/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32)).

Part of [The Cloud Native Journey](https://hune-lab.com/the-cloud-native-journey), the apiserver journey.

## The filter: no sink or no policy, no audit

[`endpoints/filters/audit.go` L43-45](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/audit.go#L43-L45):

```go
if sink == nil || policy == nil {
	return handler
}
```

## The chain: sink and policy are two fields of the server config

[`server/config.go` L1064](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/config.go#L1064), in `DefaultBuildHandlerChain`:

```go
handler = genericapifilters.WithAudit(handler, c.AuditBackend, c.AuditPolicyRuleEvaluator, c.LongRunningFunc)
```

So `sink` is `c.AuditBackend` and `policy` is `c.AuditPolicyRuleEvaluator`.

## The flags

Three flags of `server/options/audit.go` fill the options struct:

| Flag | Field | Line |
|---|---|---|
| `--audit-policy-file` | `o.PolicyFile` | [L262](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/options/audit.go#L262) |
| `--audit-log-path` | `o.LogOptions.Path` | [L440](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/options/audit.go#L440) |
| `--audit-webhook-config-file` | `o.WebhookOptions.ConfigFile` | [L547](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/options/audit.go#L547) |

## From the flags to the config: `AuditOptions.ApplyTo`

[`server/options/audit.go` L273-341](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/options/audit.go#L273-L341), shortened:

```go
// 1. Build policy evaluator
evaluator, err := o.newPolicyRuleEvaluator()

// 2. Build log backend
var logBackend audit.Backend
w, err := o.LogOptions.getWriter()
// …
if w != nil {
	if evaluator == nil {
		klog.V(2).Info("No audit policy file provided, no events will be recorded for log backend")
	} else {
		logBackend = o.LogOptions.newBackend(w)
	}
}

// 3. Build webhook backend
var webhookBackend audit.Backend
if o.WebhookOptions.enabled() {
	if evaluator == nil {
		klog.V(2).Info("No audit policy file provided, no events will be recorded for webhook backend")
	} else {
		// …
		webhookBackend, err = o.WebhookOptions.newUntruncatedBackend(nil)
	}
}

// …
// 4. Apply dynamic options.
var dynamicBackend audit.Backend
if webhookBackend != nil {
	// if only webhook is enabled wrap it in the truncate options
	dynamicBackend = o.WebhookOptions.TruncateOptions.wrapBackend(webhookBackend, groupVersion)
}

// 5. Set the policy rule evaluator
c.AuditPolicyRuleEvaluator = evaluator

// 6. Join the log backend with the webhooks
c.AuditBackend = appendBackend(logBackend, dynamicBackend)
```

Each value comes from one flag:

- **`evaluator`** reads `--audit-policy-file`. With no file, there is no policy ([L349-359](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/options/audit.go#L349-L359)):

  ```go
  func (o *AuditOptions) newPolicyRuleEvaluator() (audit.PolicyRuleEvaluator, error) {
  	if o.PolicyFile == "" {
  		return nil, nil
  	}
  	p, err := policy.LoadPolicyFromFile(o.PolicyFile)
  	// …
  	return policy.NewPolicyRuleEvaluator(p), nil
  }
  ```

- **`logBackend`** reads `--audit-log-path`. With no path, `getWriter` returns no writer ([L496-502](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/options/audit.go#L496-L502)):

  ```go
  func (o *AuditLogOptions) enabled() bool {
  	return o != nil && o.Path != ""
  }

  func (o *AuditLogOptions) getWriter() (io.Writer, error) {
  	if !o.enabled() {
  		return nil, nil
  	}
  ```

- **`dynamicBackend`** is the webhook backend, wrapped so that oversized events are truncated. It reads `--audit-webhook-config-file` ([L578-579](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/options/audit.go#L578-L579)):

  ```go
  func (o *AuditWebhookOptions) enabled() bool {
  	return o != nil && o.ConfigFile != ""
  }
  ```

## No backend at all: `appendBackend` returns nil

[`server/options/audit.go` L57-65](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/options/audit.go#L57-L65):

```go
func appendBackend(existing, newBackend audit.Backend) audit.Backend {
	if existing == nil {
		return newBackend
	}
	if newBackend == nil {
		return existing
	}
	return audit.Union(existing, newBackend)
}
```

## What it means

Audit needs two things: a policy (`--audit-policy-file`) and at least one backend (`--audit-log-path` or `--audit-webhook-config-file`). Without a policy, neither backend is built, `c.AuditBackend` stays nil, and `WithAudit` returns the next handler unchanged.

## RKE2 sets the flags for you

Some distributions pass these flags to the apiserver themselves. RKE2 is one of them. Source: **RKE2 v1.37.2+rke2r1** (commit [`972ca61add47`](https://github.com/rancher/rke2/tree/972ca61add47eabb7088d1304e523b4e8ee8b04b)).

With the `cis` profile, an empty `audit-policy-file` key becomes `/etc/rancher/rke2/audit-policy.yaml`; a path you set yourself is kept. As soon as a path is chosen, RKE2 writes a default policy there if the file does not exist, and passes `--audit-policy-file` and `--audit-log-path` to the apiserver, the log going to `<data-dir>/server/logs/audit.log` ([`pkg/executor/staticpod/staticpod.go` L228-249](https://github.com/rancher/rke2/blob/972ca61add47eabb7088d1304e523b4e8ee8b04b/pkg/executor/staticpod/staticpod.go#L228-L249)):

```go
if s.ProfileMode.isCISMode() {
	args = append([]string{"--service-account-extend-token-expiration=false"}, args...)
	if s.AuditPolicyFile == "" {
		s.AuditPolicyFile = podtemplate.DefaultAuditPolicyFile // /etc/rancher/rke2/audit-policy.yaml
	}
}

if s.AuditPolicyFile != "" {
	if err := podtemplate.WriteDefaultPolicyFile(s.AuditPolicyFile); err != nil {
		return err
	}
	extraArgs := []string{
		"--audit-policy-file=" + s.AuditPolicyFile,
		// …
	}
	if auditLogFile == "" {
		auditLogFile = filepath.Join(s.DataDir, "server/logs/audit.log")
		extraArgs = append(extraArgs, "--audit-log-path="+auditLogFile)
	}
	args = append(extraArgs, args...)
}
```

The default policy records nothing: it has a single rule, at level `None`, with no filter, so it matches every request ([`pkg/podtemplate/utils.go` L36-54](https://github.com/rancher/rke2/blob/972ca61add47eabb7088d1304e523b4e8ee8b04b/pkg/podtemplate/utils.go#L36-L54)):

```go
func WriteDefaultPolicyFile(policyFilePath string) error {
	auditPolicy := auditv1.Policy{
		// …
		Rules: []auditv1.PolicyRule{
			{
				Level: "None",
			},
		},
	}
	// …
	return writeIfNotExists(policyFilePath, bytes)
}
```

So `WithAudit` is in the chain, and yet nothing is written. Replace this file with a real policy as soon as API access must be traced, `cis` profile included. RKE2 never overwrites an existing file.

Official documentation: [Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/), [RKE2 CIS hardening](https://docs.rke2.io/security/hardening_guide).
