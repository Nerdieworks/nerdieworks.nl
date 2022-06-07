---
title: "Traefik: add trailing slash"
date: 2022-06-07T22:23:44+02:00
tags:
  - traefik
  - tutorial
  - snippet
---

When using [Traefik](https://doc.traefik.io/) as your [Kubernetes ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) you might want to add a trailing slash (`/about` → `/about/`) to prevent duplicate routes. Traefik does not (yet) support adding trailing slashes through a configuration option[<sup>[1]</sup>](https://github.com/traefik/traefik/issues/563)[<sup>[2]</sup>](https://github.com/traefik/traefik/issues/5159). You can use this snippet to create a [Traefik IngressRoute](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/) and [Traefik Middleware](https://doc.traefik.io/traefik/middlewares/http/redirectscheme/) that will do just that.

## Prerequisites

- A running instance of Traefik v2 in a Kubernetes cluster
- `kubectl` access to the Kubernetes cluster
- External IP address of the server/VM for testing

## Snippet

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-domain-com-add-trailing-slash # Change this
  namespace: default # Change this?
spec:
  entryPoints:
    - websecure # Change this?
  routes:
    - kind: Rule
      # Matches requests with a non-empty path and only ones that do not end with
      # a file extension (e.g. '.jpg') or an already existing trailing slash
      match: "Host(`www.my-domain.com`) && PathPrefix(`/{path:.+}`) && !Path(`/{path:(.+)(\\.\\w+|/)}`)" # Change this
      priority: 200
      middlewares:
        - name: add-trailing-slash
          namespace: traefik
      services:
        - kind: TraefikService
          name: noop@internal
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: add-trailing-slash
  namespace: traefik
spec:
  redirectRegex:
    permanent: true
    regex: "^https?://(.*)/(.+)"
    replacement: "https://${1}/${2}/"
```

## Usage

1. Save the snippet above to a file called `my-domain-com-add-trailing-slash.yaml`. [You can download it here](/resources/my-domain-com-add-trailing-slash.yaml)

2. Make the necessary changes

3. Create the resources:

   ```bash
   kubectl apply -f my-domain-com-add-trailing-slash.yaml
   ```

4. Try it out with:

   ```bash
   curl -v https://www.my-domain.com/foo/bar --resolve www.my-domain.com:443:{your server IP}
   # 301 Moved Permanently
   # Location: https://www.my-domain.com/foo/bar/
   ```

## Notes

In case you've enabled the `providers.kubernetesCRD.ingressClass` in the static configuration, you will have to add the [`kubernetes.io/ingress.class`](https://doc.traefik.io/traefik/providers/kubernetes-crd/#ingressclass) annotation to the `IngressRoute`.
