+++
title = "OIDC with K3s"
publishDate = "2024-12-31T08:00:00Z"
tags = ["System Administration", "SSO", "K3s", "Kubernetes", "Authentik"]
showFullContent = false
readingTime = false
hideComments = false
+++

I want to authenticate to work on my kubernetes (k3s) cluster not for any particular reason but specifically for knowledges sake. Following this I was able to get most of the way but there were a couple caveats and it doesn't seem to be documented that many places. I'm not sure if it's because it's a bare metal situation and most k8 clusters are run on hyperscalers or if it's just not something that's usually done.

Following this handy guide from FunkyPenguin I was able to get most of the way, I'll need to do a pul request and try to get it updated. [geek-cookbook.funkypenguin.co.nz/kubernetes/oidc-authentication/k3s-authentik/](https://geek-cookbook.funkypenguin.co.nz/kubernetes/oidc-authentication/k3s-authentik/)

There were a couple things that were catching me up. This might be a rehashing of the entire page, but when I had to rebuild the cluster I did get caught again and needed to do digging.

Within Authentik you'll need to create an application, provider, and group. Attach the group to your testing user. 

This will need to be added to the end of the k3s config file for all of the control-plane masters. `/etc/rancher/k3s/config.yaml`

```
kube-apiserver-arg:
  - "oidc-issuer-url=<application URL from authentik>"
  - "oidc-client-id=kube-apiserver"
  - "oidc-username-claim=email"
  - "oidc-groups-claim=groups"
  - "oidc-username-prefix=oidc:"
  - "oidc-groups-prefix=oidc:"
```

Of note the original guide does not have the last two lines defining the prefix. After the addition has been made to the control-plane masters k3s will need to be restarted `systemctl restart k3s`.

on the client devices [github.com/int128/kubelogin](https://github.com/int128/kubelogin) will need to be installed and placed into the path.

run a test using `kubectl oidc-login setup --oidc-issuer-url=<application URL from authentik> --oidc-client-id=kube-apiserver --oidc-client-secret=<your secret> --oidc-extra-scope=profile,email`

There will be several things that return. Of primary interest is the second step. Which should return a list of groups. If groups are not returned you will need to verify the groups claim is supplied in the config file and the profile scope is requested with the token.

Looking to step 5 "Set up the kubeconfig" we can run that command to finish setting up the local client. This command can be run as given with no adjustments. 

Turning back to cluster configuration we will need to create a clusterrolebinding. This will create an immutable resource binding the `admin-kube-apiserver` group (authentik) to a `cluster-admin` as this is the highest privilege and permission this is far from ideal so this will need to be pared down. But for instructional purposes and to get initially running it will suffice. When adjusting groups within Authentik if you need to change what user is logged in without waiting for a timeout to occur you can delete the directory `~/.kube/cache`.

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: oidc-group-admin-kube-apiserver
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin 
subjects:
- kind: Group
  name: oidc:admin-kube-apiserver
  ```

As I use flux I create the file and put it into the git repository and wait for it to populate. Doing a check at this point should be successful `kubectl --user=oidc get nodes`. If it works you can set it as the default using `kubectl config set-context --current --user=oidc`. If you would like to delete the default cluster-admin level of config and have a mandatory login you can delete the default user with `kubectl config delete-user default` if you have a break-glass measure to login this should be fine. 