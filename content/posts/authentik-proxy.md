+++
title = "Authentik Proxy"
publishDate = "2025-01-01T08:00:00Z"
tags = ["System Administration", "SSO", "K3s", "Kubernetes", "Authentik"]
showFullContent = false
readingTime = false
hideComments = false
+++

Setting up a page to authenticate and protect an unauthenticated page using Authentik, from a kubernetes cluster isn't as well documented if your primary ingress is nginx. Traefik seems to be more popular and better documented. Authentik does have some documentation but there are a couple clarifying steps missing. 

[docs.goauthentik.io/docs/add-secure-apps/providers/proxy/server_nginx](https://docs.goauthentik.io/docs/add-secure-apps/providers/proxy/server_nginx)

For starters, create the group that will be assigned the app, users do not need to be assigned yet. Then create an application and a provider using the Wizard. We can use the embedded outpost that's provided alongside the Authentik helm install. When supplying details on the Provider, "Advanced flow Settings->Authentication flow" should be set. While still within Authentik's admin panel we need to add the application in question to the outpost that will be used or create a new one as needed. 

This will prevent receving a 500 error as a client from the nginx logs and a 404 error from the authentik logs. Doing this would have allowed me to receive an actual informative error of not being authenticated.

Create the ingress resource. There are two ingresses in play with this setup. The first is the authentication layer and the second is the ingress attached directly to the service that needs to be protected. This ingress needs to have HTTPS enabled on it so that will need to be specified. Here's an example ingress for the outpost. In this case `app.company` is the external URL for the resource that needs to be protected. We are also poitning it at the service of the outpost we are using. 

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: authentik-outpost
  namespace: authentik
spec:
  ingressClassName: nginx
  rules:
    - host: app.company
      http:
        paths:
          - path: /outpost.goauthentik.io
            pathType: Prefix
            backend:
              # Or, to use an external Outpost, create an ExternalName service and reference that here.
              # See https://kubernetes.io/docs/concepts/services-networking/service/#externalname
              service:
                name: ak-outpost-authentik-embedded-outpost
                port:
                  number: 9000
  tls:
    - hosts:
        - app.company
      secretName: production-tls-certificate
```

Initial troubleshooting also pointed at the ingress we created as being incorrectly named. Additionally we were also not correctly setting the tls keys on the ingress. the combination of both meant that `curl -i https://app.company/outpost.goauthentik.io/ping` would not return `HTTP/2 204` as intended. This secondary ingress only functions for the `/outpost.goauthentik.io/ping` path suffix.

Almost there. Now we need to go to or create the ingress resource that directly points at the service we are needing. These lines in the documentation are 1 for 1 what needs to be put in without any notes. 

```
metadata:
  annotations:
    # This should be the in-cluster DNS name for the authentik outpost service
    # as when the external URL is specified here, nginx will overwrite some crucial headers
    nginx.ingress.kubernetes.io/auth-url: |-
      http://ak-outpost-authentik-embedded-outpost.authentik.svc.cluster.local:9000/outpost.goauthentik.io/auth/nginx
    # If you're using domain-level auth, use the authentication URL instead of the application URL
    nginx.ingress.kubernetes.io/auth-signin: |-
      https://app.company/outpost.goauthentik.io/start?rd=$scheme://$http_host$escaped_request_uri
    nginx.ingress.kubernetes.io/auth-response-headers: |-
      Set-Cookie,X-authentik-username,X-authentik-groups,X-authentik-entitlements,X-authentik-email,X-authentik-name,X-authentik-uid
    nginx.ingress.kubernetes.io/auth-snippet: |
      proxy_set_header X-Forwarded-Host $http_host;
```
