# Example using distroless

Istio support a number of base images:
- distroless - 47M
- debug - 78M

## grc.io/distroless

- base - second layer includes libc and openssl
- static
- static
- python3, 2.7
- nodejs
- java
- dotnet
- cc

-debian{9,10,11} variants

## Istio distroless content

```shell

gcrane config gcr.io/istio-testing/proxyv2:1.12-dev-distroless |jq .
# shows 10 layers - first  distroless, second our additions, last 8 small parts of istio
 "config": {
    "User": "65532",
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt",
      "ISTIO_META_ISTIO_PROXY_SHA=istio-proxy:ff44db02db5a99a7f06c31441c3a5a0f7ce9e2b4",
      "ISTIO_META_ISTIO_VERSION=1.12-alpha.eae0ae5c1c492ce59ec56f73552a808b91687d58"
    ],
    "Entrypoint": [
      "/usr/local/bin/pilot-agent"
    ],
    "WorkingDir": "/",
    "OnBuild": null
  },


$ gcrane manifest --platform linux/amd64 gcr.io/istio-testing/proxyv2:1.12-dev-distroless |jq .

 https://gcr.io/v2/istio-testing/proxyv2/blobs/sha256:c5dc4f258debef99ad7e7690d50bd879f1193553e0d36747e9626cd7ac3265f8
 https://storage.googleapis.com/artifacts.istio-testing.appspot.com/containers/images/sha256:c5dc4f258debef99ad7e7690d50bd879f1193553e0d36747e9626cd7ac3265f8

$ gcrane  blob gcr.io/istio-testing@sha256:ed95b4ae780017a8aed1e302277312b9def69adfaf61f5fe86a3cc8a626b5b50 | tar tvfz -
- /var/lib/dpkg/tzdata, netbase, base
- tzdata: usr/share/zoneinfo /usr/sbin/tzconfig
- netbase: /etc/protocols,services,rpc,ethertypes
- /etc/passwd,group, nsswitch, 
- /etc/ssl/certs/ca-certificates.crt
- base: /etc/host.conf, 
- ./lib/x86_64-linux-gnu/libc-2.31.so

Second layer:
- iptables, dpkg: libc,  


# gcrane append -f <(cd ../out/cert-ssh/bin && tar -cf - sshd)  -t gcr.io/dmeshgate/ssh-signerd/sshd:latest

```
# Distroless example

This is a basic example using the 'distroless' base image and adding an application. 

The dockerfile also includes busybox from gcr.io/distroless/base:debug - this is only for debug, you should not 
need it for production.

## Test app

The included application is the Istio 'echo' test app, found in istio/pkg/test/echo. This is used in most e2e tests 
in Istio, and will be used to run the e2e tests for CloudRun. 

It will dump request headers and information, and it supports a number of options for testing. 
