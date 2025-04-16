# Flux Bootstrapping

I attempted to deploy flux with a github app, but I failed. The documentation is lacking. This is what I did:

## Redeploy cluster

### Delete cluster

```
export kind_cluster_name=team1-dev 
sudo kind delete cluster -n "${kind_cluster_name}"
rm -v "${HOME}/.kube/kind-${kind_cluster_name}"
export KUBECONFIG=$(find "${HOME}/.kube" -maxdepth 1 -type f ! -name config | tr "\n" ":"); kubectl config view --flatten > "${HOME}/.kube/config"; yq '.users[] |= select(.name == "oidc") |= .user += {"as":"root"}' -i ~/.kube/config;
> deploy/flux-system/gotk-components.yaml
> deploy/flux-system/gotk-components.yaml
git commit -am "Truncating files"
git push github.com main
```

### Create cluster

```
sudo kind create cluster --name "${kind_cluster_name}" --kubeconfig "${HOME}/.kube/kind-${kind_cluster_name}"
sudo chown u8055:u8055 "${HOME}/.kube/kind-${kind_cluster_name}"
export KUBECONFIG=$(find "${HOME}/.kube" -maxdepth 1 -type f ! -name config | tr "\n" ":"); kubectl config view --flatten > "${HOME}/.kube/config"; yq '.users[] |= select(.name == "oidc") |= .user += {"as":"root"}' -i ~/.kube/config; chmod 600 ~/.kube/config; unset KUBECONFIG;
k config use-context "kind-${kind_cluster_name}"
```

## Flux bootstrap

1. Create a temporary PAT to bootstrap: https://github.com/settings/personal-access-tokens/new
   - Token name: flux bootstrap for team1-dev
   - Resource owner: 6j0-org
   - Repository access: only select repositories
   - Under Repository permissions select (https://fluxcd.io/flux/installation/bootstrap/github/#github-organization):
     - Administration -> Access: Read and write
     - Contents -> Access: Read and write
1. Bootstrap flux:
   ```
   flux bootstrap github \
     --owner=6j0-org \
     --repository=team1-dev \
     --branch=main \
     --path=deploy \
     --context="kind-${kind_cluster_name}" \
     --components-extra image-reflector-controller,image-automation-controller
   ```
1. Pull the changes that flux just made:
   ```
   git pull github.com main
   ```
1. TODO: Should we create a user to create the GitHub App? Whoever creates the app can delete it and affect everything that's using it.
1. Create a new GitHub App by going to https://github.com/settings/apps/new
   - GitHub App Name: fluxcd.io
   - Homepage URL: https://fluxcd.io/
   - Under Webhook, uncheck Active
   - Under Repository permissions select (https://fluxcd.io/flux/installation/bootstrap/github/#github-organization):
     - Administration -> Access: Read and write
     - Contents -> Access: Read and write
   - Where can this GitHub App be installed? Any account
1. Note the "App ID" - you will need this later, e.g., 1187455
1. Under https://github.com/settings/apps/fluxcd-io, scroll to the bottom and click "Generate a private key". This will automatically download a `.pem` file.
1. Go to https://github.com/settings/apps/fluxcd-io/installations and install the app to your organization.
   - Only select repositories: select your flux repo.
1. After installing the app, look in the URL bar - the Installation ID will be at the end of the URL, e.g., https://github.com/organizations/6j0-org/settings/installations/63073577
1. Create a flux secret: https://fluxcd.io/flux/cmd/flux_create_secret_githubapp/
   ```
   flux create secret githubapp ghapp-secret \
     --app-id=1187455 \
     --app-installation-id=63073577 \
     --app-private-key="${HOME}/Downloads/fluxcd-io.2025-04-15.private-key.pem"
   ```
1. Per https://fluxcd.io/blog/2025/02/flux-v2.5.0/#github-app-authentication-for-git-repositories, change the secretRef in your gotk-sync.yaml GitRepository object. Also, undocumented, change the `.spec.url` to the https URL for the repo (e.g., https://github.com/6j0-org/team1-dev.git), and add `.spec.provider: github`. Here's a one-liner:
   ```
   yq -i '(select(document_index == 0) | .spec.provider = "github" | .spec.secretRef.name = "ghapp-secret" | .spec.url |= sub("ssh://git@github.com","https://github.com")) // .'  deploy/flux-system/gotk-sync.yaml
   ```
1. Commit, push, and manually apply this change, because flux can't/won't manage the stuff in the flux-system directory? 
   ```
   git add deploy/flux-system/gotk-sync.yaml
   git commit -m 'Switching to GitHub App'
   git push github.com main
   k apply -f deploy/flux-system/gotk-sync.yaml
   ```
1. Delete the PAT you created. You won't need it again unless you are trying to upgrade flux, because flux can't bootstrap with a GitHub App alone yet...
1. I think you should be able to delete the SSH Deploy Key that flux created when it first bootstrapped: https://github.com/6j0-org/team1-dev/settings/keys, but when I did this things stopped working. It may not have been this - I need to try again.
1. Manually reconcile:
   ```
   flux reconcile source git flux-system && flux reconcile kustomization -n flux-system flux-system
   ```
