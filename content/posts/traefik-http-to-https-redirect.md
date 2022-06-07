---
title: "Traefik HTTP to HTTPS redirect"
date: 2022-06-07T12:51:50+02:00
tags:
  - traefik
  - security
  - snippet
---

# Traefik HTTP to HTTPS redirect

When using [Traefik](https://doc.traefik.io/) as your [Kubernetes ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) you would probably want a default HTTP to HTTPS redirect. You can use this snippet to create a [Traefik IngressRoute](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/) and [Traefik Middleware](https://doc.traefik.io/traefik/middlewares/http/redirectscheme/) that will do exactly that.

## Snippet

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: http-to-https-redirect
  namespace: traefik
spec:
  entryPoints:
    - web
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

1. Save the snippet above to a file called `http-to-https-redirect.yaml`
2. Update the `entryPoints` to your HTTP entrypoint(s). Traefik uses `web` for HTTP by default.
3. Run `kubectl apply -f http-to-https-redirect.yaml`
4. Try it out with `curl -v http://example.com/ --resolve example.com:80:{your server IP}`

## Notes

This `IngressRoute` will match **any** host with **any** path. It will return a `301 Moved Permanently`.

Using `priority: 1` ensures that another `IngressRoute` resource for the `web` entrypoint overrides the redirect. See [docs on routing priority](https://doc.traefik.io/traefik/routing/routers/#priority_1).

In case you've enabled the `providers.kubernetesCRD.ingressClass` in the static configuration, you will have to add the [`kubernetes.io/ingress.class`](https://doc.traefik.io/traefik/providers/kubernetes-crd/#ingressclass) annotation to the `IngressRoute`.
