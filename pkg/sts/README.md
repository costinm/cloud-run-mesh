# STS  (secure token service) client and server code

Extracted from Istio repository and cleaned up. The intent is to include it directly in the krun/hbone, to avoid
requiring pilot-agent for proxyless gRPC and 'uProxy' hbone mode.

STS is defined in RFC6750. Istio client is in stsclient.go (used for MeshCA) and tokenexchangeplugin.go.

Golang gRPC has credentials/sts/sts.go - unfortunately the API requires the token to be saved to a path

OAuth2 package includes downscope.NewTokenSource that wraps STS.

Stackdriver uses a similar STS exchange, implemented in Envoy, with STS server in istio-agent, using:

```json
 {
        "stackdriver_grpc_service": {
        "google_grpc": {
          "stat_prefix": "oc_stackdriver_tracer",
          "channel_credentials": {
            "ssl_credentials": {
              "root_certs": {
                "filename": "/etc/ssl/certs/ca-certificates.crt"
              }
            }
          },
          "call_credentials": {
            "sts_service": {
              "token_exchange_service_uri": "http://localhost:{{ .stsPort }}/token",
              "subject_token_path": "/var/run/secrets/tokens/istio-token",
              "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
              "scope": "https://www.googleapis.com/auth/cloud-platform",
            }
          }
        },
        "initial_metadata": [
          {
            "key": "x-goog-user-project",
            "value": "{{ .gcp_project_id }}"
          }
        ]
      },
}
```

## Generate access/ID token

[generateAccessToken](https://cloud.google.com/iam/docs/reference/credentials/rest/v1/projects.serviceAccounts/generateAccessToken)

Requires 'iam.serviceAccounts.getAccessToken' permission or roles/iam.serviceAccountTokenCreator

## Initial credentials

Identity is bootstrapped from existing platform credentials.

Sources:
- GOOGLE_APPLICATION_CREDENTIALS 
- $HOME/.config/gcloud/application_default_credentials.json
- metadata server
- $HOME/.kube/config 
- in-cluster token/CA addr/cert

The identity returned by initial credentials can be:
- a User - who might be admin on k8s.
- a GSA - with specific permissions assigned for the application. 
- a KSA

The trust domain is derived from the projectID - for gke://CONFIG_PROJECT, and for 
explicit clusters the projectId of the cluster.

Google credentials are found using golang.org/x/oauth2/google FindDefaultCredentialsWithParams().

## Federated identity

Google (and others) support 'federated' identity, where an OIDC identity provider like github
or gitea, issuing their own tokens, is marked as "trusted".

Once the IDP is trusted, the tokens are exchanged for Google tokens, which are 
access tokens encoding the foreign identity. 

The foreign identity can be used in IAM for authz directly - but not in all apps.
It can also be used to impersonate a regular GSA - getting access or OIDC tokens
for the specified GSA.

```shell
gcloud iam service-accounts add-iam-policy-binding \
  GSA_NAME@GSA_PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:WORKLOAD_IDENTITY_POOL[K8S_NAMESPACE/KSA_NAME]"
```

Some google APIs - like Mesh CA, CA service, managed Istiod - require the 
federated token associated with the GKE IDP and KSA. The use it to extract 
the namespace and KSA name.

Envoy can get tokens to authenticate with the XDS server using STS protocol,
metadata server (MDS) or GOOGLE_APPLICATION_CREDENTIALS (GAC)

Proxyless gRPC can get tokens using MDS or GAC.

To get the app or envoy to use the right tokens, we can:
- capture MDS, via iptables or env variable override
- configure a GAC 

An example, from [Fleet setup](https://cloud.google.com/anthos/multicluster-management/fleets/workload-identity#-go)

```json
{
      "type": "external_account",
      "audience": "identitynamespace:${CONFIG_PROJECT_ID}.svc.id.goog:IDENTITY_PROVIDER",
      "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/GSA_NAME@GSA_PROJECT_ID.iam.gserviceaccount.com:generateAccessToken",
      "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
      "token_url": "https://sts.googleapis.com/v1/token",
      "credential_source": {
        "file": "/var/run/secrets/tokens/gcp-ksa/token"
      }
    }

```

IDENTITY_PROVIDER can be projects/${CONFIG_PROJECT_NUMBER}/locations/global/memberships/MEMBERSHIP
for fleet-registered projects, or ...

The file must be refreshed periodically outside K8S, and must have the 
audience ${CONFIG_PROJECT_ID}.svc.id.goog

