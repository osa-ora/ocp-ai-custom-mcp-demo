# OCP AI Custom MCP Server Demo
Demo to show how to deploy our custom HR MCP Server over OpenShift.

This is how it looks when this MCP Server deployed into OpenShift:

<img width="1536" height="1024" alt="HR-MCP-Server-OCP" src="https://github.com/user-attachments/assets/562d4372-7fde-4867-ad39-a38490d72e33" />

## Steps to Run the Demo on OpenShift Environment

### 1) Create Project and Configurations

After login to OpenShift cluster, follow these steps:

```bash
# Create new project
oc new-project hr-mcp

# Create ConfigMap
oc create configmap hr-mcp-config \
--from-literal=DATABASE_USER=postgres \
--from-literal=DATABASE_HOST=postgresql.hr-mcp.svc.cluster.local \
--from-literal=DATABASE_PORT=5432 \
--from-literal=DATABASE_NAME=hrdb \
--from-literal=MCP_SCHEME=http \
--from-literal=MCP_HOST=0.0.0.0 \
--from-literal=MCP_PORT=8080 \
--from-literal=MCP_PATH=/mcp \
--from-literal=MCP_TRANSPORT=http \
--from-literal=DEBUG=true \
-n hr-mcp

# Or
oc apply -f https://raw.githubusercontent.com/osa-ora/ocp-ai-custom-mcp-demo/refs/heads/main/ocp/configmap.yaml -n hr-mcp

# Create Secret
oc create secret generic hr-mcp-secret \
  --from-literal=DATABASE_USER=postgres \
  --from-literal=DATABASE_PASSWORD=postgres \
  -n hr-mcp

# Or
oc apply -f https://raw.githubusercontent.com/osa-ora/ocp-ai-custom-mcp-demo/refs/heads/main/ocp/secret.yaml -n hr-mcp

```
### 2) Deploy Our HR DB and initialize it

Execute the following command (or you can provision it from the OCP GUI):

```
oc new-app postgresql-persistent \
  --param=POSTGRESQL_USER=postgres \
  --param=POSTGRESQL_PASSWORD=postgres \
  --param=POSTGRESQL_DATABASE=hrdb \
  --name=postgresql -n hr-mcp
```

Login to the DB Pod and run the following commands: 

```
psql -U postgres

CREATE DATABASE hrdb;

\q
```

Then install the DB schema and test data from inside the Pod:

```
curl -s https://raw.githubusercontent.com/osa-ora/ocp-ai-custom-mcp-demo/refs/heads/main/db_scripts/schema.sql \
| psql -U postgres -d hrdb

curl -s https://raw.githubusercontent.com/osa-ora/ocp-ai-custom-mcp-demo/refs/heads/main/db_scripts/load_sample_data.sql \
| psql -U postgres -d hrdb
```

### 3) Provision the mcp_server using S2I

Provision it and attach the configMap and Secret
```

oc new-app python:3.12-minimal-ubi10~https://github.com/osa-ora/ocp-ai-custom-mcp-demo --name=hr-mcp-server -n hr-mcp

oc set env deployment/hr-mcp-server --from=configmap/hr-mcp-config
oc set env deployment/hr-mcp-server --from=secret/hr-mcp-secret
oc rollout restart deployment/hr-mcp-server

```

if started successfully, check the logs to confirm that no issues and end points are correctly mapped: 

<img width="1169" height="570" alt="Screenshot 2026-06-11 at 12 11 33 AM" src="https://github.com/user-attachments/assets/744a4fa9-a9ae-487a-9f5c-49b0d163e686" />

### 4) Testing the MCP Server:

If you want to test it remotely then expose a route:

```
oc expose svc/hr-mcp-server -n hr-mcp
```

if you need to test it from inside OpenShift using the playground, then go to project redhat-ods-applications and edit the ConfigMap: gen-ai-aa-mcp-servers by adding the following section (Service endpoint/mcp) :

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: gen-ai-aa-mcp-servers
  namespace: redhat-ods-applications
  ....
  ....
data:
  HR-MCP-Server: |
    {
      "url": "http://hr-mcp-server.hr-mcp.svc.cluster.local:8080/mcp",
      "description": "An MCP server for interacting with our HR DB tools."
    }
```
Or you can create MCPServer to auto-discovered using the OpenShift AI MCP server section

Something like this: 
```
oc apply -f - <<EOF
apiVersion: mcp.opendatahub.io/v1alpha1
kind: MCPServer
metadata:
  name: my-hr-mcp-server
  namespace: hr-mcp
spec:
  image: image-registry.openshift-image-registry.svc:5000/test/hr-mcp-server@sha256:549f50c109f4c6d040e948edf3169fc811be7559ec171f66062fa06f0ad60929
  transportType: sse
  port: 8000
  envFrom:
    - configMapRef:
        name: hr-mcp-config
    - secretRef:
        name: hr-mcp-secret
EOF
```
But i didn't tested it, as i don't have the MCP Operstor enabled in my environment.

Now go to OpenShift UI, check the MCP Servers section:

<img width="1466" height="704" alt="Screenshot 2026-06-11 at 12 05 18 AM" src="https://github.com/user-attachments/assets/19924c77-fde4-4593-87ae-412bae7c7dcb" />

Then create a playground for the MCP server and one of the existing models, and enjoy chatting to it: 

<img width="1477" height="721" alt="Screenshot 2026-06-11 at 12 04 44 AM" src="https://github.com/user-attachments/assets/1c8f1742-cf05-439d-8dd9-acf3e00e55d7" />

Try to modify the prompt to give the LLM guidelines how to use the HR MCP server, for example: "if you don't have the employee_code try to get it first before calling other functions."


To connect to this KCP server remotely from outside OpenShift, for example: ChatBox application, get the route URL or execute the following command.

```
oc get route hr-mcp-server -n hr-mcp -o jsonpath='{.spec.host}{"\n"}'
```
Add the http prefix and the path configured /mcp at the end, and configure it in ChatBox or any other applicaton as following: 

<img width="767" height="682" alt="Screenshot 2026-06-11 at 12 07 04 AM" src="https://github.com/user-attachments/assets/2e7f2a26-11d7-4a40-97e9-bf27a714ae4a" />

And enjoy chatting with our HR MCP Server...

<img width="763" height="685" alt="Screenshot 2026-06-11 at 7 59 21 AM" src="https://github.com/user-attachments/assets/ccca83bb-7e29-418d-8c8e-84b80e569792" />

Example requests:

- leave balance for Osama Oransa?
- basic profile for Sara Ali
- Show my full profile for EMP001?
- policy for remote work?
- leave requests for Osama Oransa.
- basic profile for EMP002

### Note: To deploy a sample OpenShift MCP Client App, follow the following guide: https://github.com/osa-ora/ocp-ai-mcp-client-demo



