# Flux 2 Testing

This repo was created from https://github.com/fluxcd/flux2-kustomize-helm-example/tree/main.

It uses this as well: https://fluxcd.io/flux/guides/image-update/

1. [Opt into fine-grained personal access tokens for your organization](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization)
1. [Enable deploy keys for this repo](https://github.com/organizations/6j0-org/settings/deploy_keys)
1. Create a [Fine-grained personal access token](https://github.com/settings/tokens?type=beta) with full access to this git repository.
1. Create some environment variables:
   ```bash
   export GITHUB_TOKEN=put_your_token_here
   export kind_cluster_name="team1-dev"
   export flux2_branch="main"
   if [[ "$OSTYPE" == "darwin"* ]]; then
     export sops_dir="${HOME}/Library/Application Support/sops/age"
   elif [[ "$OSTYPE" == "linux"* ]]; then
     export sops_dir="${HOME}/.config/sops/age"
   fi
   ```
1. Create a kind cluster:
   ```bash
   kind create cluster --name "${kind_cluster_name}" --kubeconfig "${HOME}/.kube/kind-${kind_cluster_name}"
   ```
1. Set your KUBECONFIG:
   ```bash
   export KUBECONFIG="${HOME}/.kube/kind-${kind_cluster_name}"
   ```
1. Bootstrap the new cluster:
   ```bash
   flux bootstrap github \
       --components-extra image-reflector-controller,image-automation-controller \
       --context="kind-${kind_cluster_name}" \
       --owner=6j0-org \
       --repository=${kind_cluster_name} \
       --branch="${flux2_branch}" \
       --path=deploy \
       --read-write-key
   ```
1. TODO: Create a webhook so ghcr.io notifies flux immediately when a new image is published. See https://fluxcd.io/flux/guides/image-update/#trigger-image-updates-with-webhooks
   ```
   TOKEN=$(head -c 12 /dev/urandom | shasum | cut -d ' ' -f1)
   kubectl -n flux-system create secret generic webhook-token --from-literal=token=$TOKEN
   ```
1. Download keys.txt from 1password - you can find it in the K8s repo, named "sops keys.txt".
1. Move it to the appropriate directory (see https://github.com/getsops/sops?tab=readme-ov-file#22encrypting-using-age):
   > WARNING: If you already have a keys.txt file in your sops age directory, you should probably skip this step.
   ```bash
   mkdir -p "${sops_dir}"
   mv -n "${HOME}/Downloads/keys.txt" "${sops_dir}"
   chmod 600 "${sops_dir}/keys.txt"
   ```
1. Encrypt your secrets, and delete the cleartext secret:
   > WARNING: keys.txt is unencrypted, and it contains private keys! This is dangerous. Unfortunately, sops doesn't currently work with encrypted age keys files - see https://github.com/getsops/sops/issues/933 (give it a thumbs up to bring more attention to this issue). Despite this bug, I think security is more important than convenience in this case, so I'm recommending that we encrypt our key.txt. See the "Using SOPS" section below for a workaround to this issue.
   ```bash
   age -p "${sops_dir}/keys.txt" > "${sops_dir}/keys.age"
   rm "${sops_dir}/keys.txt"
   ```
1. Add the SOPS age key so the cluster can decrypt secrets:
   ```bash
   age -d "${sops_dir}/keys.age" | kubectl create secret generic sops-age --namespace=flux-system --from-file=age.agekey=/dev/stdin
   ```
1. Add this decryption block to clusters/test/apps.yaml (see yq command below to add it as a one-liner):
   > NOTE: This only needs to be done when this repo is first created. You can skip this step now.
   ```yaml
   spec:
     decryption:
       provider: sops
       secretRef:
         name: sops-age
   ```
   yq (version 4) command that does this:
   ```bash
   yq eval '.spec.decryption = {"provider": "sops", "secretRef": {"name": "sops-age"}}' -i clusters/test/apps.yaml
   ```

Flux should now work in your kind cluster, and you should see a running dashboard-ui-app pod like this:

```bash
$ k get pods -n dashboard-ui
NAME                               READY   STATUS    RESTARTS   AGE
dashboard-ui-app-99bccfbcb-tdtmh   1/1     Running   0          5m14s
```

# Using SOPS

Because we have encrypted our age key, we have to use this command to use `sops`. You will have to enter your age key password every time you use the `sops` command. This is the burden of being responsible...

- Linux
  ```bash
  SOPS_AGE_KEY=$(age -d "${HOME}/.config/sops/age/keys.age") sops your_filename_here
  ```
- Mac
  ```bash
  SOPS_AGE_KEY=$(age -d "${HOME}/Library/Application Support/sops/age/keys.age") sops your_filename_here
  ```

Don't put that `SOPS_AGE_KEY` in your rc or profile file - that's worse than having a cleartext file on your drive. That env variable must be used exactly as specified above so that it only exists within the scope of running that sops command.
