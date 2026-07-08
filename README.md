# create-customer

An IBM App Connect Enterprise (ACE) integration that exposes a modern REST API and mediates it to a legacy SOAP customer-creation web service — the worked example our App Connect pipeline compiles into a deployable BAR.

Part of the IBM Client Engineering **Cloud Pak for Integration production-deployment demo**: this repo is the ACE *integration source* — the "what" that the App Connect build pipeline packages and GitOps promotes onto the cluster.

## What this is

A single, self-contained integration: a REST client accepts a JSON customer, ACE maps it to a SOAP request, calls the backend `CustomerDetails` web service, maps the SOAP response back to JSON, and returns it — with an explicit, deterministic fault path for backend errors. It is intentionally small and realistic: enough moving parts (flows, subflows, graphical maps, supporting Java, externalized config, tests) to exercise a production ACE build without business noise.

## What's inside

- **`createcustomer/`** — the ACE REST API application project:
  - `openapi.json` — OpenAPI 3.0.1 contract; `POST /createcustomer` with `customer_details`, `customer_details_response`, and `fault` schemas.
  - `restapi.descriptor` + `gen/createcustomer.msgflow` — the generated REST API message flow (WSInput/WSReply, RouteToLabel) that binds the OpenAPI operation to its implementation, with JSON fault format.
  - `postCreatecustomer.subflow` — the operation logic: a JavaCompute node, JSON→SOAP map, SOAP Request node, SOAP Extract, and HTTP header handling.
  - `postCreatecustomer_JSONtoSOAP.map`, `postCreatecustomer_ToJSON.map`, `postCreatecustomer_fault_mapping.map` — the three graphical maps (request in, response out, fault out).
  - `customerDetails.wsdl` / `*.xsd` / `IBMdefined/` — the backend SOAP contract (`CustomerDetailsPortSoap11`) and imported SOAP/XML schemas.
- **`createcustomerJava/`** — `SetUpDesitination.java`, a JavaCompute node that reads the backend URL from a `UserDefined` policy (`backendurl` → `soapEndpoint`) and builds the LocalEnvironment `Destination` the SOAP Request node uses.
- **`createcustomer_Test/`** — ACE JUnit tests for each map (JSON→SOAP, SOAP→JSON, fault mapping) with input/output fixtures.
- **`jmeter_test_plan/`** — JMeter plans (plus a Prometheus-reporting variant) for load and performance testing.
- **`test/newman/test.json`** — a Postman/Newman collection for API smoke tests (200 status, `success`, UUID `customerId`).
- **`pipeline_properties.yaml`** — packaging metadata consumed by the build pipeline (projects to compile, application/release names, license, base image, version).

## Why it's built this way

- **REST-in, SOAP-out mediation is the core value.** Consumers get a clean, contract-first JSON/OpenAPI interface while the legacy SOAP backend is integrated unchanged — no caller ever has to know SOAP exists. This is exactly the modernization story ACE is bought to tell.
- **Contract-first, low-code maps.** The API is generated from `openapi.json` and the transforms are graphical maps, not hand-rolled parsing — so the integration is auditable and maintainable, and the OpenAPI doc *is* the consumer contract.
- **Externalized endpoint = one artifact, many environments.** The backend URL lives in a policy fed from a secret/configuration, never baked into the flow. The same compiled BAR promotes across dev/test/prod untouched, with environment-specific config injected at deploy time — the 12-factor / GitOps discipline that makes promotion a reviewable pull request.
- **An explicit fault path.** SOAP faults are mapped deterministically to a JSON `fault` schema with a `400`, giving consumers a stable error contract instead of leaked backend stack traces.
- **Tests at three levels** (map unit tests, Newman API checks, JMeter load) let the pipeline gate the BAR before it ever reaches the cluster.

> **Note on `pipeline_properties.yaml`:** it still describes the older `integrationServer` packaging model and pins now-dated ACE versions (e.g. `12.0.1.0-r3`) and a non-production license. The current pipeline targets the 2026 IntegrationRuntime CR model and current base images; treat this file as the example's original, un-modernized state.

## How it fits the bigger picture

This repo is a **source input**, not a runtime. The App Connect build pipeline runs `ibmint` over the `createcustomer` and `createcustomerJava` projects to compile a **BAR**, containerizes it on the ACE server base image, and deploys it onto Cloud Pak for Integration via GitOps — the same rails the MQ and other integration assets ride. It derives from the [cloud-native-toolkit ACE example](https://production-gitops.dev/guides/cp4i/ace/cloud-native-example/example/) and is adapted for our CP4I pipelines. Together with the MQ and ACE infrastructure factories and the GitOps application overlays, it demonstrates the full path from an integration developer's project to a governed, auditable production deployment.

Maintained by IBM Client Engineering.
