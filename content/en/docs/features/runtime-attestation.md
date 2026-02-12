---
title: Runtime Attestation
date: 2025-01-08
description: Measurement from workload at runtime
categories:
- feature 
tags:
- attestation 
---
Workloads can request runtime attestation of arbitrary data via a generic interface.
Not all hardware platforms support runtime attestation, but those that do
will fulfill the request.

On these platforms, the Attestation Agent maintains an event log,
which tracks attestation events. This log will be forwarded to Trustee
via the KBS protocol and compared to the hardware evidence.

### Enabling

To enable this feature, set the following parameter in the guest kernel command line.
```bash
agent.guest_components_rest_api=all
```
{{% alert title="Warning" color="primary" %}}
Note that this configuration will also allow the workload to fetch attestation
reports at runtime. In some configurations this can be dangerous.
See [attestation report](get-attestation) page for more information.
{{% /alert %}}

The Attestation Agent configuration must also have runtime attestation enabled.
This can be set via Init-Data, with the `[eventlog_config]` section below.
This configuration can also specify the measurement index that will be used.
On platforms with limited indices, the index will be mapped to the available
registers. For example, on TDX setting the measurement index to 17 will usually
result in extending RTMR 3.

```toml
version = "0.1.0"
algorithm = "sha384"
[data]
"aa.toml" = '''
[token_configs]
[token_configs.kbs]
url = "http://<trustee-uri>"

[eventlog_config]
init_pcr = 17
enable_eventlog = true
'''

"cdh.toml" = '''
[kbc]
name = "cc_kbc"
url = "http://<trustee-uri>"
'''
```

The above configurations can be added to a workload as annotations.
See the [Init-Data](initdata) page for more information on using Init-Data.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        io.katacontainers.config.hypervisor.kernel_params: "agent.guest_components_rest_api=all"
	io.katacontainers.config.hypervisor.cc_init_data: "H4sIAAAAAAAAA4WOwQ6DIBBE7/sVhos3xGpSa9IvMYQgUiEiGFz9/kLTHtpLjzO7M29OHXcbfHEvCKM1ZQSkm0O0aNbs7UY2XUtgmCRKDkRKimF1JN3KsoQBw6K9UME/7LzzH02XMXlHdLnJIG59VdWMXtqWMnpr+o51iQeDPrVHF+Z3joP1FsWmYsrVV9Bejk6Lz1cyMR4aMh+Imsz3omVUHLxcdYYqJZImfze8up7xdiRhCwEAAA=="
    spec:
      runtimeClassName: (...)
      containers:
      - name: nginx
        (...)
```
### Runtime Attestation

Once enabled, runtime attestation can be triggered via the rest API.
```bash
curl -X POST http://127.0.0.1:8006/aa/aael \
     -H "Content-Type: application/json" \
     -d '{"domain":"test","operation":"test","content":"test"}'
```

The `domain` and `operation` are context fields that will be included in the event log.

You can check that the event log was updated by inspecting an attestation token.
```bash
curl http://127.0.0.1:8006/aa/token\?token_type\=kbs | jq -r '.token |split(".") | .[1] | @base64d | fromjson'
```
The event log may contain boot-time entries, but at the end you should see your entry.

```json
{
   "details":{
      "data":{
         "content":"test",
         "domain":"test",
         "operation":"test"
      },
      "string":"test test test",
      "unicode_name":"AAEL"
   },
   "digest_matches_event":true,
   "digests":[
      {
         "alg":"SHA-384",
         "digest":"1495be3eb2120e59facb8f92447d64f..."
      }
   ],
   "event":"TEVBQREAAAB0ZXN0IHRlc3QgdGVzenp6dA==",
   "index":4,
   "type_name":"EV_EVENT_TAG"
}
```
