## Basic setup

1. Install kind
2. Create 2 clusters
```bash
kind create cluster --name cluster-a
kind create cluster --name cluster-b
```
3. On cluster-b
```bash
# Configure spire server
kubectx kind-cluster-b
kubectl apply -f spire-namespace.yaml
kubectl apply -f server-account.yaml
kubectl apply -f spire-bundle-configmap.yaml # what's this for?
kubectl apply -f server-cluster-role.yaml
kubectl apply -f server-configmap.yaml
kubectl apply -f server-statefulset.yaml
kubectl apply -f server-service.yaml

# Configure spire agents
kubectl apply -f agent-account.yaml
kubectl apply -f agent-cluster-role.yaml
kubectl apply -f agent-configmap.yaml
kubectl apply -f agent-daemonset.yaml
```

4. Check that agent is attested and note down the SPIFEE ID
```
kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server agent list
```


7. Create registration entry on SPIRE server
```bash
kubectl exec -n spire spire-server-0 -- \
    /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.org/test-workload/hello-world \
    -parentID spiffe://example.org/spire/agent/k8s_sat/kind-cluster-b/20654bb0-4d0b-4729-8cff-28383474fe2e \
    -selector k8s:ns:test-workload -selector k8s:container-name:hello-world
```

8. Create the workload
```bash
# Note that it mounts the node agent socket via hostPath
kubectl apply -f test-workload.yml
```

9. Go into the pod and download the SPIFFE helper which will be used to obtain the SPIFEE ID
```bash
# Exec into pod
kubectl exec -it -n test-workload <POD_NAME> -- /bin/sh

# Download SPIFFE helper
wget https://github.com/spiffe/spiffe-helper/releases/download/v0.6.0/spiffe-helper-v0.6.0.tar.gz -O helper.tar.gz
tar -xvf helper.tar.gz

# Configure SPIFFE helper
cat << EOF > helper.conf
agentAddress = "/run/spire/sockets/agent.sock"
cmd = "sleep"
cmdArgs = "36000"
certDir = "certs"
renewSignal = "SIGUSR1"
svidFileName = "svid.pem"
svidKeyFileName = "svid_key.pem"
svidBundleFileName = "svid_bundle.pem"
EOF

mkdir -p certs

# Run 
./spiffe-helper

# Using another shell
kubectl exec -it -n test-workload <POD_NAME> -- /bin/sh

# See that X.509-SVID documents are obtained in certs
ls certs 

# If you run openssl x509 -in svid.pem -noout -text, you'll see the SPIFEE ID spiffe://example.org/test-workload/hello-world in SAN
```

## Testing Envoy integration
```
# to create a node alias
./create-node-registration-entry.sh

kubens default && kubectl apply -k envoy-x509/k8s

# You should see "target SdsApi spiffe://example.org/ns/default/sa/default/backend initialized"
kubectl logs <BACKEND_POD_NAME> envoy 
```

3. `kubectl`

### Notes
1. PSAT can be used to attest nodes (agents) that do not belong in the same cluster at the server. [Reference](https://spiffe.io/docs/latest/deploying/configuring/#service-account-tokens)
2. Each node's agent will get the SPIFEE ID of `spiffe://<trust_domain>/spire/agent/k8s_sat/<cluster>/<UUID>`. 
    - See `create-node-registration-entry.sh` to create a node alias for all nodes such that all nodes have the same SPIFEE ID
    - See the **Mapping Workloads to Multiple Nodes** section under https://spiffe.io/docs/latest/deploying/registering 
3. The Envoy-OPA SPIRE integration gels quite nicely with our RAPID setup
4. `hostPath` mounting is a must to for workloads to access the node's agent
5. Agent's SPIFEE ID changes on restart
   1. Could be a `kind` thing
### Painpoints
1. Painful to create registration entries 1 by 1

### Questions
1. [How does outgoing traffic work for Envoy](https://spiffe.io/docs/latest/microservices/envoy-x509/readme/)
1. Service mesh vs DIY SPIRE, which is better?
1. Security SIEM use cases and how to enable them

### Next steps
1. [How to use envoy sidecar to handle SPIFEE communication instead](https://github.com/spiffe/spire-tutorials/tree/main/k8s/envoy-x509)
2. How to authorise
3. Test with workload outside k8s
