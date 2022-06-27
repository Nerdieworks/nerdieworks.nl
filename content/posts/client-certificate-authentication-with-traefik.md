---
title: "Client certificate authentication with Traefik"
date: 2022-06-27T21:15:17+02:00
tags:
  - traefik
  - tutorial
  - snippet
  - security
---

Sometimes authentication with a username and password is not secure enough. A login form might contain vulnerabilities or give away important information, like what software you are running on the given domain (and what version of it). **Client certificate authorization** is just like using SSH: your client has a private and public key installed that it uses to let the server know it can be trusted (through [mutual TLS](https://www.cloudflare.com/en-gb/learning/access-management/what-is-mutual-tls/)). It's not very user friendly, but it IS a great way to secure your web app.

Doing this with [Traefik](https://doc.traefik.io/traefik/) is fairly straightforward:

1. Generate a certificate authority (CA)
2. Add the CA certificate to Traefik
3. Create an [IngressRoute](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/) that uses the CA certificate
4. Generate a client certificate
5. Install the client certificate on your client(s)

## 1. Generate a certificate authority (CA)

Generate the CA certificate with the `openssl` command:

```bash
openssl genrsa -des3 -out 'ca.key' 4096
openssl req -x509 -new -nodes -key 'ca.key' -sha384 -days 1825 -out 'ca.crt'
```

**Important:** store the `ca.key` in a secure place.

Next, create a `Secret` in the same namespace where the `IngressRoute` will be in:

```bash
kubectl create secret generic 'client-auth-ca-cert' -n 'my-namespace' --from-file='ca.crt=ca.crt'
```

Make sure the key is either `ca.crt` or `tls.ca`. See the [TLSOption documentation](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-tlsoption) for more info.

## 2. Add the CA certificate to Traefik

Create a `TLSOption` in the same namespace as the `Secret` that we have created in the previous step:

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: TLSOption
metadata:
  name: client-cert # Change this
  namespace: my-namespace # Change this
spec:
  minVersion: VersionTLS12
  maxVersion: VersionTLS13
  clientAuth:
    secretNames:
      - client-auth-ca-cert
    clientAuthType: RequireAndVerifyClientCert
  curvePreferences:
    - CurveP521
    - CurveP384
  cipherSuites:
    - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
    - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
    - TLS_AES_256_GCM_SHA384
    - TLS_CHACHA20_POLY1305_SHA256
  sniStrict: true
```

At this point the CA certificate is ready to use.

## 3. Create an IngressRoute that uses the CA certificate

Use the following snippet. Make changes where necessary.

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: example-com # Change this
  namespace: my-namespace # Change this
spec:
  entryPoints:
    - websecure # Change this?
  routes:
    - kind: Rule
      match: "Host(`example.com`)" # Change this
      services:
        # You can use this service for testing. It will respond with a '418 I'm a teapot'
        - kind: TraefikService
          name: noop@internal
  tls: # Not merged with static configuration
    # certResolver: letsencrypt # You can add this later
    options:
      # Change these. Must match the metadata from step 2
      name: client-cert
      namespace: my-namespace
```

Once created, try `curl -vk https://example.com --resolve example.com:443:{server IP}`. It should respond with a `bad certificate` error.

## 4. Generate a client certificate

Now that everything is set up on the server side, we can create a client key and certificate:

```bash
export DOMAIN=example.com # Same as Host in IngressRoute's match
export ORGANIZATION=Example
export COUNTRY=NL

openssl genrsa -des3 -out "$DOMAIN.key" 4096
openssl req -new -key "$DOMAIN.key" -out "$DOMAIN.csr" -subj "/CN=$DOMAIN/O=$ORGANIZATION/C=$COUNTRY"

# Use ca.crt and ca.key from step 1
openssl x509 -sha384 -req -CA 'ca.crt' -CAkey 'ca.key' -CAcreateserial -days 365 -in "$DOMAIN.csr" -out "$DOMAIN.crt"
openssl pkcs12 -export -out "$DOMAIN.pfx" -inkey "$DOMAIN.key" -in "$DOMAIN.crt"
```

Let's try out the certificate:

```bash
curl -vk https://example.com --resolve example.com:443:{server IP} --cert "$DOMAIN.crt" --key "$DOMAIN.key"
```

The server should now respond with a `418 I'm a teapot`.

## 5. Install the client certificate on your client(s)

Most clients support a way to add a client certificate. The best way to figure out how is to [do a quick online search](https://duckduckgo.com/?q=install+client+certificate+firefox). Here are some examples:

- Add to **Firefox**: Settings > Privacy & Security > View Certificates > Your Certificates > Import.
- Add to **Android**: Settings > Security > Advanced > Encryption & Credentials > Install from SD card (tested in Chrome).
- Add to Firefox on Android: didn't get it to work, despite trying https://blog.jeroenhd.nl/article/firefox-for-android-using-a-custom-certificate-authority.

When connecting to your domain for the first time, the client usually shows a popup to select the installed client certificate.

That's it. You have now established a very secure connection with your server. Just make sure you don't leak one of your private key files!
