---
title: Routing North-South Traffic into an Istio Mesh with Application Gateway for Containers
date: 2026-06-01T14:12:47+00:00
author: Mike Hacker
tags:
- App Modernization
- Security
- How To
categories:
- Azure
- Kubernetes
summary: A technical walkthrough for combining Application Gateway for Containers with the AKS Istio add-on to deliver encrypted, mTLS-protected ingress for containerized government workloads.
draft: false
image_prompt: A bright, sunlit parcel-sorting atrium where colorful sealed packages move from a wide-open exterior doorway onto one broad conveyor, then pass a warm reception counter where two matching hand-pressed wax tokens are compared before the parcels split into tidy lanes across an open interior courtyard. Golden-hour light pours through tall windows, emphasizing the clear front-door path, orderly routing, and trusted handoff into the mesh-like network of lanes. No text, letters, numbers, or writing anywhere in the image.
audio: audio.mp3
image: cover.png
---

Containerized applications running on Azure Kubernetes Service (AKS) usually have two traffic paths to secure. The first is **east-west** traffic, which is the service-to-service communication inside the cluster. The second is **north-south** traffic, which is the traffic arriving from outside the cluster and needs a secure, scalable front door. A service mesh such as Istio is designed for the first problem. Application Gateway for Containers is designed for the second. This post walks through how to combine them into an encrypted, identity-aware ingress pattern that platform teams can operate with clear ownership boundaries.

Application Gateway for Containers is the evolution of the Application Gateway Ingress Controller and is part of the Application Gateway product family. The Istio-based service mesh add-on for AKS is a Microsoft-supported, tested integration of upstream Istio. The Application Gateway for Containers overview was updated on April 24, 2026, and documents Layer 7 routing, Gateway API v1 support, ports 80 and 443, Web Application Firewall (WAF), and mutual authentication at the frontend, the backend, or end to end ([overview](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/overview)).

## The architecture pattern

The clean way to wire these together is to let each component own what it does best:

1. **Application Gateway for Containers** terminates the public TLS connection, applies WAF policy when configured, and performs Layer 7 routing based on hostname, path, header, query string, or method. It lives outside the cluster data plane and is responsible for ingress.
2. It forwards traffic to the **Istio ingress gateway** service inside the cluster. For a stronger boundary, configure backend mutual TLS so Application Gateway for Containers presents a client certificate and validates the certificate served by the Istio ingress gateway.
3. The **Istio mesh** then enforces identity-based, mTLS-encrypted communication for east-west calls between meshed workloads, with no application code changes.

The result is encryption on each hop from the internet edge to the workload. Application Gateway for Containers authenticates to the mesh edge, the Istio ingress gateway validates that client identity, and the mesh authenticates workload-to-workload calls. That layered model maps directly to zero trust principles: never trust the network, verify identity, and encrypt traffic in transit.

## Standing up the ingress layer

Application Gateway for Containers offers two deployment strategies. The **managed by ALB Controller** strategy lets the in-cluster ALB Controller create and manage the Azure resources for you. The **bring-your-own (BYO)** strategy provisions the Azure resources independently through tools such as CLI, PowerShell, Bicep, or Terraform and references them from Kubernetes. BYO is often the better fit for government platform teams because it keeps infrastructure lifecycle under change control.

The AKS managed add-on is the fastest path to a working controller. Complete the current quickstart prerequisites first, including provider registration, required CLI extensions, and feature registration where the documentation lists it. On a new cluster:

```bash
az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_NAME \
    --location $LOCATION \
    --network-plugin azure \
    --enable-oidc-issuer \
    --enable-workload-identity \
    --enable-gateway-api \
    --enable-application-load-balancer \
    --generate-ssh-keys
```

The add-on requires Azure CNI or Azure CNI Overlay, workload identity, the AKS Gateway API add-on, and a supported AKS Kubernetes version. The quickstart also notes that the Application Gateway for Containers add-on requires the AKS Gateway API add-on to avoid conflicts with other Gateway API consumers ([quickstart](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/quickstart-deploy-application-gateway-for-containers-alb-controller)). Verify the controller is healthy before going further:

```bash
kubectl get pods -n kube-system | grep alb-controller
```

For BYO deployments that you want to express as infrastructure as code, the Application Gateway for Containers parent resource is `Microsoft.ServiceNetworking/trafficControllers`. A minimal Bicep definition using the stable API version looks like this:

```bicep
param resourceName string = 'agfc-gov-prod'
param location string = 'westus'

resource trafficController 'Microsoft.ServiceNetworking/trafficControllers@2023-11-01' = {
  name: resourceName
  location: location
}
```

The template reference shows `2023-11-01` as the stable (non-preview) API version and `2025-03-01-preview` as the latest preview version. The advanced security wiring lives in the preview schema: `2025-03-01-preview` exposes `securityPolicyConfigurations.wafSecurityPolicy` and `securityPolicyConfigurations.ipAccessRulesSecurityPolicy` ([Bicep reference](https://learn.microsoft.com/en-us/azure/templates/microsoft.servicenetworking/trafficcontrollers)). If you would rather not hand-author the frontend, association, and managed identity wiring, the [Azure Verified Module for the Traffic Controller](https://github.com/Azure/bicep-registry-modules/tree/main/avm/res/service-networking/traffic-controller) encapsulates the resource set.

## Defining the Gateway

With the controller running, you declare ingress using Gateway API resources. Application Gateway for Containers implements `GatewayClass`, `Gateway`, `HTTPRoute`, and `ReferenceGrant`; the overview notes that `ReferenceGrant` support is currently v1alpha1. The listener ports are restricted to 80 and 443. A TLS-terminating HTTPS Gateway under the BYO strategy is defined like this:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-01
  namespace: istio-system
  annotations:
    alb.networking.azure.io/alb-id: $RESOURCE_ID
spec:
  gatewayClassName: azure-alb-external
  listeners:
  - name: https-listener
    port: 443
    protocol: HTTPS
    allowedRoutes:
      namespaces:
        from: Same
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        group: ''
        name: frontend.com
  addresses:
  - type: alb.networking.azure.io/alb-frontend
    value: $FRONTEND_NAME
```

Confirm the Gateway is accepted, the listener reaches `Programmed` status, and a frontend FQDN is assigned before attaching routes ([Gateway API how-to](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/how-to-ssl-offloading-gateway-api)). Your `HTTPRoute` should point at the Istio ingress gateway service rather than directly at application pods. If the route and the Istio ingress service are in different namespaces, add the required Gateway API cross-namespace reference controls for your design.

## Closing the loop with backend mTLS

The security-critical piece is the hop between Application Gateway for Containers and the mesh. There are two sides to configure:

1. Application Gateway for Containers needs a `BackendTLSPolicy` so it presents a client certificate and validates the Istio ingress gateway's server certificate.
2. The Istio ingress gateway needs an Istio `Gateway` configured with mutual TLS so it requires and validates the client certificate presented by Application Gateway for Containers.

Application Gateway for Containers documents backend mutual TLS through the `BackendTLSPolicy` resource. The policy can target the Istio ingress gateway service:

```yaml
apiVersion: alb.networking.azure.io/v1
kind: BackendTLSPolicy
metadata:
  name: istio-ingress-tls-policy
  namespace: aks-istio-ingress
spec:
  targetRef:
    group: ''
    kind: Service
    name: aks-istio-ingressgateway-external
    namespace: aks-istio-ingress
  default:
    sni: ingress.contoso.gov
    ports:
    - port: 443
    clientCertificateRef:
      name: gateway-client-cert
      group: ''
      kind: Secret
    verify:
      caCertificateRef:
        name: ca.bundle
        group: ''
        kind: Secret
      subjectAltName: ingress.contoso.gov
```

The `clientCertificateRef` is the certificate Application Gateway for Containers presents to the Istio ingress gateway. The `verify` block is what Application Gateway for Containers uses to confirm that it is talking to the genuine mesh edge and not an impostor ([backend mTLS how-to](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/how-to-backend-mtls-gateway-api), [Kubernetes API specification](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/api-specification-kubernetes)).

Then configure the receiving Istio ingress gateway to require a client certificate. The AKS Istio secure gateway guidance shows this by setting TLS mode to `MUTUAL` and using a Kubernetes secret that contains `tls.key`, `tls.crt`, and `ca.crt` in the `aks-istio-ingress` namespace:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: agfc-mtls-gateway
  namespace: aks-istio-ingress
spec:
  selector:
    istio: aks-istio-ingressgateway-external
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: MUTUAL
      credentialName: ingress-gateway-credential
    hosts:
    - ingress.contoso.gov
```

Use an Istio `VirtualService` bound to that Istio `Gateway` to route requests to the application service. After traffic is inside the mesh, use `PeerAuthentication` in `STRICT` mode for namespaces or workloads that must reject plaintext traffic:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: workloads
spec:
  mtls:
    mode: STRICT
```

Istio defines `STRICT` as requiring an mTLS tunnel for peer authentication, and the AKS Istio add-on documents secure service-to-service communication with TLS encryption, strong identity-based authentication, and authorization ([AKS Istio add-on](https://learn.microsoft.com/en-us/azure/aks/istio-about), [PeerAuthentication reference](https://istio.io/latest/docs/reference/config/security/peer_authentication/)).

## Hardening the network boundary

The association subnet that connects Application Gateway for Containers into your virtual network supports full Network Security Group control for associations created on or after April 23, 2026. Both inbound and outbound NSG rules are honored, including deny-all rules with explicit allow exceptions. For associations created before April 23, 2026, inbound ports 80 and 443 are always permitted regardless of configured inbound rules ([components reference](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/application-gateway-for-containers-components)).

For agencies that route through a network virtual appliance, user-defined routes on the association subnet are supported. The important caveat is that internet ingress must return to the internet to avoid asymmetric routing.

## Why This Matters for Government

State and local agencies modernizing legacy systems into containers inherit a tougher compliance bar than many commercial teams. NIST SP 800-53 provides a catalog of security and privacy controls for information systems and organizations, and CISA's Zero Trust Maturity Model describes zero trust as a shift away from location-centric trust toward granular, per-request access decisions. This pattern supports those goals with public TLS termination and optional WAF at a managed Azure edge, mutual TLS into the mesh, and identity-based mTLS for internal calls. The application does not need to implement that certificate choreography itself ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), [CISA Zero Trust Maturity Model](https://www.cisa.gov/resources-tools/resources/zero-trust-maturity-model)).

Offloading the ingress data plane to a managed Azure service also reduces operational burden on lean public-sector platform teams. Application Gateway for Containers provides autoscaling, availability-zone resilience, and near real-time updates as pods, routes, and probes change. The Istio add-on adds Microsoft-tested compatibility with supported AKS versions and Microsoft-managed lifecycle operations for Istio components when upgrades are triggered by the user.

One planning note specific to public-sector deployments: many agencies run workloads across both Azure commercial and Azure Government. The Application Gateway for Containers overview lists supported regions, and that list currently covers commercial regions such as East US, West US 2, and North Europe. Before committing to a design in an Azure Government region, confirm current regional availability against the [supported regions list](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/overview#supported-regions) and your account team because feature parity between clouds can lag. Validate the same for the AKS Istio add-on in your target region.

## Next steps

If you are already running the Istio add-on on AKS, the incremental work to adopt this pattern is manageable: enable the Application Gateway for Containers add-on, define a Gateway and HTTPRoute that target the Istio ingress gateway service, attach a `BackendTLSPolicy`, and configure the Istio ingress gateway itself for `MUTUAL` TLS. Start in a non-production cluster, confirm the Gateway reaches `Programmed`, confirm the route reports `Accepted`, and test both sides of the mTLS handshake before promoting the configuration through your normal change process. The official quickstarts and how-to guides linked throughout this post are the authoritative references to build from.
