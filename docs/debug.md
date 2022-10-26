# Troubleshooting

Env variables to get more debug:


| Key             | Value     | Description |
|-----------------|-----------|---| 
| FORCE_START     | 1         | Ignore errors, keep running to the end. SSH to the instance to debug startup issues |
| XDS_AGENT_DEBUG | sds:debug | debug logs for startup auth errors. Other agent options possible |

# Commands

```shell
curl -vvv -k  --cert /var/run/secrets/istio.io/cert-chain.pem --key /var/run/secrets/istio.io/key.pem -cacert /var/run/secrets/istio.io/root-cert.pem  https://10.4.1.54:8080/fortio/ 
openssl s_client -showcerts -connect 10.4.1.52:8080
openssl x509 -text -in /var/run/secrets/istio.io/cert-chain.pem 


 ssh -v -p 15022  10.128.0.157 iptables-save -c
` ssh -v -p 15022  $IP curl localhost:15000/config_dump
`


```
