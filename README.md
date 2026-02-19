# Testing MCP backends: context7 and github-mcp

This file documents quick, repeatable steps to verify the two MCP backends deployed in this workspace:
- `context7` 
- `github-mcp` 

Prerequisites
- `kubectl` configured to target your local cluster
- `npx` (node) available for the inspector CLI

## install envoy gateway

```sh
helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-gateway-system \
  --create-namespace \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml

kubectl wait --timeout=2m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```
## install envoy ai gateway CRDs
```sh
helm upgrade -i aieg-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace
  ```
## install envoy ai gateway resources
```sh
helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace

kubectl wait --timeout=2m -n envoy-ai-gateway-system deployment/ai-gateway-controller --for=condition=Available
```
## install aigw-core and model charts
from here https://github.com/ADORSYS-GIS/ai-helm/tree/main/charts/ai-gateway-core

1) Ensure secrets

Replace `YOUR_KEY` with the real API key if you have one. For testing you can use `test-key` for context7.

```bash
# context7 secret
kubectl -n converse-llm create secret generic context7-token --from-literal=apiKey=YOUR_KEY --dry-run=client -o yaml | kubectl apply -f -

# brave-search secret
kubectl -n converse-llm create secret generic github-mcp-token --from-literal=apiKey=<PAT-here> --dry-run=client -o yaml | kubectl apply -f -

```

2) Ensure Gateway accepts routes from other namespaces (only needed if your Gateway listener restricts namespaces):

```bash
kubectl -n default patch gateway ai-gateway --type='json' -p='[{"op":"replace","path":"/spec/listeners/0/allowedRoutes","value":{"namespaces":{"from":"All"}}}]'
```

3) Port-forward the Envoy gateway locally (find the service name then forward)

```bash
export ENVOY_SERVICE=$(kubectl get svc -n envoy-gateway-system \
  --selector=gateway.envoyproxy.io/owning-gateway-namespace=default,gateway.envoyproxy.io/owning-gateway-name=ai-gateway \
  -o jsonpath='{.items[0].metadata.name}')

kubectl port-forward -n envoy-gateway-system svc/$ENVOY_SERVICE 8080:80

# set GATEWAY_URL for local testing
export GATEWAY_URL="http://localhost:8080"
```

4) Use the inspector to list MCP tools (context7 should appear)

```bash
npx @modelcontextprotocol/inspector@0.16.8 --cli "$GATEWAY_URL/mcp" --method tools/list
```

- Expected: tool definitions for context7 and github-mcp
