---
title: "Traefik: non-WWW to WWW redirect"
date: 2022-06-07T21:55:36+02:00
tags:
  - traefik
  - tutorial
  - snippet
---

When using [Traefik](https://doc.traefik.io/) as your [Kubernetes ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) you might want to redirect any requests to your top-level domain (for instance `my-domain.com`) to its WWW variant (`www.my-domain.com`). You can use this snippet to create a [Traefik IngressRoute](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/) and [Traefik Middleware](https://doc.traefik.io/traefik/middlewares/http/redirectscheme/) that will do just that.

## Prerequisites

- A running instance of Traefik v2 in a Kubernetes cluster
- `kubectl` access to the Kubernetes cluster
- External IP address of the server/VM for testing

## Snippet

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-domain-com-www-redirect # Change this
  namespace: default # Change this?
spec:
  entryPoints:
    - websecure # Change this?
  routes:
    - kind: Rule
      match: Host(`my-domain.com`) # Change this
      middlewares:
        - name: redirect-to-www
          namespace: traefik
      services:
        - kind: TraefikService
          name: noop@internal
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect-to-www
  namespace: traefik
spec:
  redirectRegex:
    permanent: true
    regex: "^https?://(?:www\\.)?(.+)"
    replacement: "https://www.${1}"
```

## Usage

1. Save the snippet above to a file called `my-domain-com-www-redirect.yaml`. [You can download it here](/resources/my-domain-com-www-redirect.yaml)

2. Make the necessary changes

3. Create the resources:

   ```bash
   kubectl apply -f my-domain-com-www-redirect.yaml
   ```

4. Try it out with:

   ```bash
   curl -v https://my-domain.com/ --resolve my-domain.com:443:{your server IP}
   # 301 Moved Permanently
   # Location: https://www.my-domain.com/
   ```

## Notes

In case you've enabled the `providers.kubernetesCRD.ingressClass` in the static configuration, you will have to add the [`kubernetes.io/ingress.class`](https://doc.traefik.io/traefik/providers/kubernetes-crd/#ingressclass) annotation to the `IngressRoute`.
