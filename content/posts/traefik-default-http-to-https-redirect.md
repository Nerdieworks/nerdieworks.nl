---
title: "Traefik: default HTTP to HTTPS redirect"
date: 2022-06-07T12:51:50+02:00
tags:
  - traefik
  - tutorial
  - snippet
  - security
---

When using [Traefik](https://doc.traefik.io/) as your [Kubernetes ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) you would probably want a default HTTP to HTTPS redirect. You can use this snippet to create a [Traefik IngressRoute](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/) and [Traefik Middleware](https://doc.traefik.io/traefik/middlewares/http/redirectscheme/) that will do just that.

## Prerequisites

- A running instance of Traefik v2 in a Kubernetes cluster
- `kubectl` access to the Kubernetes cluster
- External IP address of the server/VM for testing

## Snippet

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: http-to-https-redirect
  namespace: traefik
spec:
  entryPoints:
    - web # Change this?
  routes:
    - kind: Rule
      match: PathPrefix(`/`)
      priority: 1
      middlewares:
        - name: redirect-to-https
      services:
        - kind: TraefikService
          name: noop@internal
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect-to-https
  namespace: traefik
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

## Usage

1. Save the snippet above to a file called `http-to-https-redirect.yaml`. [You can download it here](/resources/http-to-https-redirect.yaml)

2. Make the necessary changes

3. Create the resources:

   ```bash
   kubectl apply -f http-to-https-redirect.yaml
   ```

4. Try it out with:

   ```bash
   curl -v http://example.com/ --resolve example.com:80:{your server IP}
   # 301 Moved Permanently
   # Location: https://example.com
   ```

## Notes

This `IngressRoute` will match **any** host with **any** path.

Using `priority: 1` ensures that other `IngressRoute` resources for the `web` entrypoint override the redirect. See [docs on routing priority](https://doc.traefik.io/traefik/routing/routers/#priority_1).

In case you've enabled the `providers.kubernetesCRD.ingressClass` in the static configuration, you will have to add the [`kubernetes.io/ingress.class`](https://doc.traefik.io/traefik/providers/kubernetes-crd/#ingressclass) annotation to the `IngressRoute`.
