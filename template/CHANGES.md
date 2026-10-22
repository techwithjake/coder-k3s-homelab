# Template changes

Base: the Kubernetes (Deployment) starter template in Coder, plus the
`code-server` module (`modules.tf`, `registry.coder.com/coder/code-server/coder` 1.5.0).

Put your own `main.tf`, `modules.tf`, and `.terraform.lock.hcl` in this folder.
Commit the lock file so provider versions stay pinned.

## My edit to main.tf

Added a parameter above `data "coder_workspace" "me" {}`:

```hcl
data "coder_parameter" "env_label" {
  name         = "env_label"
  display_name = "Environment label"
  description  = "Shown inside the workspace as ENV_LABEL"
  default      = "dev"
  mutable      = true
  option {
    name  = "Dev"
    value = "dev"
  }
  option {
    name  = "Test"
    value = "test"
  }
}
```

And an `env` block in the `container` of `kubernetes_deployment_v1.main`:

```hcl
env {
  name  = "ENV_LABEL"
  value = data.coder_parameter.env_label.value
}
```

## Verified

After publishing and updating a workspace with the value set to Test:

```
kubectl exec -n coder <workspace-pod> -- env | grep ENV_LABEL
ENV_LABEL=test
```

## Things in the starter template worth knowing

- `strategy { type = "Recreate" }`: an update deletes the old workspace pod before the new one starts (a brief outage by design, unlike a rolling update).
- `wait_for_rollout = false`: Terraform reports success as soon as the Deployment object exists, not when the pod is healthy.
- Publishing a template version only runs `terraform init` and a plan. The apply happens when a workspace is built or updated.
