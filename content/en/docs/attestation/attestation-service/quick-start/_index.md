---
title: Quick Start
description: Quick Start to deploy the Attestation Service
weight: 2
categories:
- docs
tags:
- docs
- attestation-service
---

For the latest deployment document, please refer to the CoCoAS [github page](https://github.com/confidential-containers/trustee/tree/main/attestation-service#quick-start).

The CoCoAS supports different APIs. Now we have gRPC version and RESTful version.

## gRPC CoCo AS

### Quick Start and Test

Users can use a [community version of gRPC CoCoAS image](https://github.com/confidential-containers/trustee/pkgs/container/staged-images%2Fcoco-as-grpc) to deploy quickly.

```shell
# run gRPC CoCoAS server locally
docker run -d \
  -v <path-to-attestation-service>/docs/sgx_default_qcnl.conf:/etc/sgx_default_qcnl.conf \ # this qcnl config is used when verifying SGX/TDX quotes
  -p 50004:50004 \
  ghcr.io/confidential-containers/staged-images/coco-as-grpc:latest \
  grpc-as
```

The `sgx_default_qcnl.conf` configures the PCS (Provisioning Certification Service) / PCCS (Provisioning Certification Cache Service) of Intel platforms. If you want to verify SGX/TDX quotes, please refer to the [Intel Platform Configuration](#intel-platform-configuration) section.


We can use a sample gRPC client to test the gRPC API using the following test request body.
```json
{
    "tee": "snp",
    "evidence": "eyJhdHRlc3RhdGlvbl9yZXBvcnQiOnsidmVyc2lvbiI6MiwiZ3Vlc3Rfc3ZuIjo0LCJwb2xpY3kiOjE5NjYzOSwiZmFtaWx5X2lkIjpbMSwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMF0sImltYWdlX2lkIjpbMiwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMF0sInZtcGwiOjAsInNpZ19hbGdvIjoxLCJjdXJyZW50X3RjYiI6eyJib290bG9hZGVyIjozLCJ0ZWUiOjAsIl9yZXNlcnZlZCI6WzAsMCwwLDBdLCJzbnAiOjgsIm1pY3JvY29kZSI6MjA2fSwicGxhdF9pbmZvIjoxLCJfYXV0aG9yX2tleV9lbiI6MCwiX3Jlc2VydmVkXzAiOjAsInJlcG9ydF9kYXRhIjpbMjM2LDEwOCw4MiwyMTUsODMsNjAsMTk0LDE5NiwyNDQsOTEsMjMxLDEzMiwxNTYsMjQxLDE4LDE3MSwxMzAsMTc4LDAsMTU5LDIzMSwxODksNjcsMjMxLDMwLDIwOCwxNDAsMjAsNjQsMTAsMjE1LDIyNiwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDBdLCJtZWFzdXJlbWVudCI6WzE2MSwyNDMsMTQ3LDQsMTksMzYsMTIzLDE3OSwxNDAsMjUyLDIzLDIxLDEyMSwyMzQsNjAsMTgsMjEzLDI1NCw3MywxLDI0MCwxOTksMTQ2LDI0Niw2MywyMTUsOTMsMTUyLDI0MSwyMzksMTMwLDEyNCwzNSw4MCw2LDY4LDIyNCwyMzAsMTQ2LDIzMCwxOTAsMTQ1LDEyNywxNDQsODAsMjExLDIxMSwxNDBdLCJob3N0X2RhdGEiOlswLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDBdLCJpZF9rZXlfZGlnZXN0IjpbMyw4NiwzMyw4OCwxMzAsMTY4LDM3LDM5LDE1NCwxMzMsMTc5LDAsMTc2LDE4Myw2NiwxNDcsMjksMTcsNTksMjQ3LDIyNyw0NSwyMjIsNDYsODAsMjU1LDIyMiwxMjYsMTk5LDY3LDIwMiw3MywzMCwyMDUsMjE1LDI0Myw1NCwyMjAsNDAsMTY2LDIyNCwxNzgsMTg3LDg3LDE3NSwxMjIsNjgsMTYzXSwiYXV0aG9yX2tleV9kaWdlc3QiOlswLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMF0sInJlcG9ydF9pZCI6WzU2LDk0LDE4NiwxMjksMzMsMTA5LDIyOCwxMTksMTAxLDcyLDI1MiwxODQsMTExLDE0MiwxNzMsMywxOTMsMjM1LDIwMSw0Myw5OCw3LDI0MywzMywxMywxNTYsMjA2LDE4NywxMzcsMjAxLDE0NCw1XSwicmVwb3J0X2lkX21hIjpbMjU1LDI1NSwyNTUsMjU1LDI1NSwyNTUsMjU1LDI1NSwyNTUsMjU1LDI1NSwyNTUsMjU1LDI1NSwyNTUsMjU1LDI1NSwyNTUsMjU1LDI1NSwyNTUsMjU1LDI1NSwyNTUsMjU1LDI1NSwyNTUsMjU1LDI1NSwyNTUsMjU1LDI1NV0sInJlcG9ydGVkX3RjYiI6eyJib290bG9hZGVyIjozLCJ0ZWUiOjAsIl9yZXNlcnZlZCI6WzAsMCwwLDBdLCJzbnAiOjgsIm1pY3JvY29kZSI6MTE1fSwiX3Jlc2VydmVkXzEiOlswLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMF0sImNoaXBfaWQiOlsxOTUsMTMyLDM5LDE2MywxMyw3NiwxMjIsMjQ5LDIxNywxMTEsMTIyLDIxLDE4NSwxMTQsMTA1LDEzMCw5MCwxMDAsMjAzLDExOCwxNjIsNTMsNDcsMjUzLDkzLDI0LDE3LDkzLDEzNywxNzMsNzEsNjMsMTQyLDE0MCwxMSwyMDUsMTU0LDkzLDE0NiwxMzQsOTcsNDMsMTczLDc0LDE3MywyNTEsNjgsMzgsMzIsOTAsNTksMTU4LDc5LDIzNCwxMzAsNDgsMTcsNTMsMTYxLDExMiwyMjgsMTE5LDgyLDc4XSwiY29tbWl0dGVkX3RjYiI6eyJib290bG9hZGVyIjozLCJ0ZWUiOjAsIl9yZXNlcnZlZCI6WzAsMCwwLDBdLCJzbnAiOjgsIm1pY3JvY29kZSI6MTE1fSwiY3VycmVudF9idWlsZCI6NCwiY3VycmVudF9taW5vciI6NTIsImN1cnJlbnRfbWFqb3IiOjEsIl9yZXNlcnZlZF8yIjowLCJjb21taXR0ZWRfYnVpbGQiOjQsImNvbW1pdHRlZF9taW5vciI6NTIsImNvbW1pdHRlZF9tYWpvciI6MSwiX3Jlc2VydmVkXzMiOjAsImxhdW5jaF90Y2IiOnsiYm9vdGxvYWRlciI6MywidGVlIjowLCJfcmVzZXJ2ZWQiOlswLDAsMCwwXSwic25wIjo4LCJtaWNyb2NvZGUiOjExNX0sIl9yZXNlcnZlZF80IjpbMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDBdLCJzaWduYXR1cmUiOnsiciI6WzYsMjM1LDIyMCw3OSw3OCw2NSw2NywyMDQsOTgsMjU0LDIxLDE4NSwyNDIsMjA5LDIzNiw0NSw4NCwyMTIsMTcxLDIzLDEwMiwxNTgsODEsNDAsMzQsMjIsMjIsOTQsMTc5LDI3LDk1LDg5LDIyNSw5OCwxLDE3MCwyMjAsMTY0LDI1MSwyMjAsMjE3LDY1LDI0MSw1MCwxMDQsNTcsOCw4MCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMF0sInMiOls2NCw5OSwxMjAsMjEyLDI2LDM4LDk4LDYwLDkxLDE3MywxNTQsMTg0LDIwNiwxNTIsMjE0LDIwNSw0OSw2NywxNDQsNDMsMTQ1LDEwNywxOTksMTYzLDUyLDE4OCwyMDksMTA2LDEyOSwyMTQsMTk5LDIwLDE2MSw0OCw4NiwxNjcsMTQ2LDIwLDE4MSwxODgsODUsMTEyLDI0OSwxODEsMjAsOTMsMjA3LDIyOCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMF0sIl9yZXNlcnZlZCI6WzAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMCwwLDAsMF19fSwiY2VydF9jaGFpbiI6W3siY2VydF90eXBlIjoiVkNFSyIsImRhdGEiOls0OCwxMzAsNSw3Niw0OCwxMzAsMiwyNTEsMTYwLDMsMiwxLDIsMiwxLDAsNDgsNzAsNiw5LDQyLDEzNCw3MiwxMzQsMjQ3LDEzLDEsMSwxMCw0OCw1NywxNjAsMTUsNDgsMTMsNiw5LDk2LDEzNCw3MiwxLDEwMSwzLDQsMiwyLDUsMCwxNjEsMjgsNDgsMjYsNiw5LDQyLDEzNCw3MiwxMzQsMjQ3LDEzLDEsMSw4LDQ4LDEzLDYsOSw5NiwxMzQsNzIsMSwxMDEsMyw0LDIsMiw1LDAsMTYyLDMsMiwxLDQ4LDE2MywzLDIsMSwxLDQ4LDEyMyw0OSwyMCw0OCwxOCw2LDMsODUsNCwxMSwxMiwxMSw2OSwxMTAsMTAzLDEwNSwxMTAsMTAxLDEwMSwxMTQsMTA1LDExMCwxMDMsNDksMTEsNDgsOSw2LDMsODUsNCw2LDE5LDIsODUsODMsNDksMjAsNDgsMTgsNiwzLDg1LDQsNywxMiwxMSw4Myw5NywxMTAsMTE2LDk3LDMyLDY3LDEwOCw5NywxMTQsOTcsNDksMTEsNDgsOSw2LDMsODUsNCw4LDEyLDIsNjcsNjUsNDksMzEsNDgsMjksNiwzLDg1LDQsMTAsMTIsMjIsNjUsMTAwLDExOCw5NywxMTAsOTksMTAxLDEwMCwzMiw3NywxMDUsOTksMTE0LDExMSwzMiw2OCwxMDEsMTE4LDEwNSw5OSwxMDEsMTE1LDQ5LDE4LDQ4LDE2LDYsMyw4NSw0LDMsMTIsOSw4Myw2OSw4Niw0NSw3NywxMDUsMTA4LDk3LDExMCw0OCwzMCwyMywxMyw1MCw1MSw0OCw0OSw1MCw1Miw0OSw1NSw1Myw1Niw1MCw1NCw5MCwyMywxMyw1MSw0OCw0OCw0OSw1MCw1Miw0OSw1NSw1Myw1Niw1MCw1NCw5MCw0OCwxMjIsNDksMjAsNDgsMTgsNiwzLDg1LDQsMTEsMTIsMTEsNjksMTEwLDEwMywxMDUsMTEwLDEwMSwxMDEsMTE0LDEwNSwxMTAsMTAzLDQ5LDExLDQ4LDksNiwzLDg1LDQsNiwxOSwyLDg1LDgzLDQ5LDIwLDQ4LDE4LDYsMyw4NSw0LDcsMTIsMTEsODMsOTcsMTEwLDExNiw5NywzMiw2NywxMDgsOTcsMTE0LDk3LDQ5LDExLDQ4LDksNiwzLDg1LDQsOCwxMiwyLDY3LDY1LDQ5LDMxLDQ4LDI5LDYsMyw4NSw0LDEwLDEyLDIyLDY1LDEwMCwxMTgsOTcsMTEwLDk5LDEwMSwxMDAsMzIsNzcsMTA1LDk5LDExNCwxMTEsMzIsNjgsMTAxLDExOCwxMDUsOTksMTAxLDExNSw0OSwxNyw0OCwxNSw2LDMsODUsNCwzLDEyLDgsODMsNjksODYsNDUsODYsNjcsNjksNzUsNDgsMTE4LDQ4LDE2LDYsNyw0MiwxMzQsNzIsMjA2LDYxLDIsMSw2LDUsNDMsMTI5LDQsMCwzNCwzLDk4LDAsNCwxOTgsOTcsMTgxLDEwMSwxODcsMTY4LDEsMiwxODksMjIxLDY4LDE0NSwyMDEsMTQ4LDI4LDE3OSw0MiwyNywxMjUsMTgyLDEyOCwxOCwxMzAsMTMyLDE2LDE4MywyNTUsMTQwLDE3MywyNTMsMTEyLDIyOSw3MywxODMsOTEsMTIwLDE3OSwyMDUsMjE0LDkyLDIwNSwyMzUsMTY4LDEzNCwyMTAsMjM4LDE2MSwyMTIsMjksMTIsNjMsMjAsMTA4LDE0MiwxODksMjE0LDEzMiw4MiwyMDYsMTI2LDE5NSwxMiwxMDUsOSwxMDMsMTk1LDE1OCw5OCw3NiwxLDE1LDE1NiwxODIsNiwxMDYsMTI4LDQ5LDEwLDEzNSw4MywxMDYsMTQ4LDIzNSwxNzQsNDEsMTk0LDE3MCwyMTcsMTI4LDIyLDE5LDE1MSwxOSwzMSwxODcsNywxNjMsMTMwLDEsMjIsNDgsMTMwLDEsMTgsNDgsMTYsNiw5LDQzLDYsMSw0LDEsMTU2LDEyMCwxLDEsNCwzLDIsMSwwLDQ4LDIzLDYsOSw0Myw2LDEsNCwxLDE1NiwxMjAsMSwyLDQsMTAsMjIsOCw3NywxMDUsMTA4LDk3LDExMCw0NSw2Niw0OCw0OCwxNyw2LDEwLDQzLDYsMSw0LDEsMTU2LDEyMCwxLDMsMSw0LDMsMiwxLDMsNDgsMTcsNiwxMCw0Myw2LDEsNCwxLDE1NiwxMjAsMSwzLDIsNCwzLDIsMSwwLDQ4LDE3LDYsMTAsNDMsNiwxLDQsMSwxNTYsMTIwLDEsMyw0LDQsMywyLDEsMCw0OCwxNyw2LDEwLDQzLDYsMSw0LDEsMTU2LDEyMCwxLDMsNSw0LDMsMiwxLDAsNDgsMTcsNiwxMCw0Myw2LDEsNCwxLDE1NiwxMjAsMSwzLDYsNCwzLDIsMSwwLDQ4LDE3LDYsMTAsNDMsNiwxLDQsMSwxNTYsMTIwLDEsMyw3LDQsMywyLDEsMCw0OCwxNyw2LDEwLDQzLDYsMSw0LDEsMTU2LDEyMCwxLDMsMyw0LDMsMiwxLDgsNDgsMTcsNiwxMCw0Myw2LDEsNCwxLDE1NiwxMjAsMSwzLDgsNCwzLDIsMSwxMTUsNDgsNzcsNiw5LDQzLDYsMSw0LDEsMTU2LDEyMCwxLDQsNCw2NCwxOTUsMTMyLDM5LDE2MywxMyw3NiwxMjIsMjQ5LDIxNywxMTEsMTIyLDIxLDE4NSwxMTQsMTA1LDEzMCw5MCwxMDAsMjAzLDExOCwxNjIsNTMsNDcsMjUzLDkzLDI0LDE3LDkzLDEzNywxNzMsNzEsNjMsMTQyLDE0MCwxMSwyMDUsMTU0LDkzLDE0NiwxMzQsOTcsNDMsMTczLDc0LDE3MywyNTEsNjgsMzgsMzIsOTAsNTksMTU4LDc5LDIzNCwxMzAsNDgsMTcsNTMsMTYxLDExMiwyMjgsMTE5LDgyLDc4LDQ4LDcwLDYsOSw0MiwxMzQsNzIsMTM0LDI0NywxMywxLDEsMTAsNDgsNTcsMTYwLDE1LDQ4LDEzLDYsOSw5NiwxMzQsNzIsMSwxMDEsMyw0LDIsMiw1LDAsMTYxLDI4LDQ4LDI2LDYsOSw0MiwxMzQsNzIsMTM0LDI0NywxMywxLDEsOCw0OCwxMyw2LDksOTYsMTM0LDcyLDEsMTAxLDMsNCwyLDIsNSwwLDE2MiwzLDIsMSw0OCwxNjMsMywyLDEsMSwzLDEzMCwyLDEsMCwyLDEyOCwzOCwxNjIsMjQ3LDMxLDMsMSwxMDgsMjE1LDI1NSw5OCwzMCwxNDgsMjEzLDE2NiwxMzgsMjE5LDEzMiw1LDQwLDE3MCwyNDQsNDcsOTQsMTQsMTEyLDY4LDExNCw2OCwyNCwxMzgsNjQsMzMsNjEsMTcxLDMxLDEwNiw5LDIzMiw4OCw4MCw0NSw0MiwyMzksMjE3LDg5LDUwLDEzNSwxMzksMjI0LDc2LDExMCwxOCwxNzYsMTc5LDAsODIsMjQxLDEwOSw2LDIxNSw0NCwzNCwxMTMsNjksMTM0LDE1MSw1NiwxNjAsMzUsMTM5LDkzLDE5OSwyMywyNDUsOTYsMTgsMTE0LDEwLDEzMiwyMTAsNTQsMjAzLDE4LDEwOCwxNjksMTM2LDEzNSwxNTIsMjIyLDE1MiwyMywyMzUsMTg4LDEyOCwxMDQsMjE1LDMzLDI5LDI0OSwyMzgsMTIyLDEwMCwxNDcsMjksMTMyLDIyMywzMCwyNTEsMjEsMTQ4LDExMCwyNTAsNDcsODAsNDUsMTkxLDIzNiw1NywxMjMsMjMzLDI1MiwxOTIsMTA0LDAsMTM5LDc0LDEzOCwyMTcsODIsMjU0LDg3LDYwLDE1NiwxMCw5NSwxLDE0LDMwLDE0MiwxOTcsMzMsMTk2LDY4LDE0MiwxMzQsMTAzLDI0OSwyNDIsMTYzLDM3LDU3LDIzMCwxMTcsMTE5LDMwLDIwOCwxNzYsMjUwLDI0NSwxNywyMzUsMjUwLDE5MSwxNTYsMTIzLDMzLDU5LDI0Niw5LDEzOCwyMjMsODYsMjAwLDI0NCw1NCwzNCwzOCwxMzAsMjQ4LDQ1LDIyNSwxNTcsOTMsMTU3LDIyNCw0OCwyMjksNjcsNzEsODMsMTY3LDE0LDEzOSw1NywxNDgsMjI0LDkyLDg3LDIzNCwxNzQsMzksMTYyLDExMCwxNDIsMTUyLDcsMTE5LDIxNiw1NSw3MywxNzQsMTk1LDE4NywxODYsMTEwLDE3Niw2Myw2OSwxMTcsMTcwLDEyNSwyNDIsMTM1LDI1LDE3OCw4NCw5NSwyMiw0OSw5OCwyNTUsMjUzLDE1MiwxMTcsODMsNTYsNDgsMTY2LDIzNCwyMzIsMTQwLDQxLDk2LDIwOCwxMjYsMjE2LDI0MCwxMzIsNDAsMjA4LDE4Nyw3MywyNDYsMjA2LDU0LDYsODgsMzgsMjI3LDIxNywxNTYsMjA3LDI0MCw4NiwzMywxOCwyNTMsMTk1LDI1MSw0OSwxNDcsNzMsMjEyLDE2NSwxMCw0LDE0MywxMTQsMTM1LDE2NCwyMzAsMTQ5LDQzLDI1LDEwMCwyMzEsMzcsMjQsMTU1LDIzNSw1OCwxOTIsODksMTM4LDEwMCwyMTUsMTY0LDQsMjI3LDExNiwyMTEsNDUsMTE0LDkxLDEzOCwxOTQsMTYwLDIwLDgzLDU0LDE2LDE1Myw3OCwyMTksMTI3LDEwMCwyMDQsMTE2LDIyOSwyNDIsMywyMTYsMTM1LDIzMSwyLDQxLDE0Myw4OSwyMiwyNCw3Nyw4NSwxODQsODAsMyw3NiwxNjIsMTEsMTQ1LDU1LDU4LDQ3LDUwLDI0Myw1Nyw5NiwxNjgsMTI0LDE4OCw5NSwxNjIsMTIwLDgzLDE4MSwzMiwyMzcsMjA0LDEzOSwyNTUsMTg1LDIzMywxMjEsMjI3LDE3NywxNTMsMTcwLDE3NSwxNzEsODUsMjcsMjMwLDM0LDM5LDEzMiwxODQsNzUsMTk1LDE3LDYwLDQ2LDE2MiwxMzcsMzEsODcsMTcwLDE3NCwyMjAsNjMsMjUzLDQyLDIyOCwxNzEsMjQ3LDI0LDE3OSwxNTgsOTcsMzIsMTk5LDQxLDIzOCwxMTksMjQ3LDI0OSwyMTksMTUyLDIzMywxNjQsMTQ3LDI4LDE1LDI1MCwzMSwxMDksMTIsODQsMTAyLDIyNSwxMzgsMTg5LDE4OSwyMjMsMjIzLDYwLDE5MCwxMDQsMTg4LDEwNiwxMzcsMjE4LDIzLDIzOCw3LDI2LDIxNywxMTksMTI1LDE0MywzNyw1MCwyMjksMTQzLDI1MiwyMjMsMjMwLDc3LDExMSw2MiwxNSwxMDMsMzEsNTMsODAsMTU2LDIxNSw4NCwxNDMsNjQsNCw3MiwxMjQsMjU1LDI0LDE2NSwxLDU2LDExNywxMDMsMTksMTU4LDYxLDQ4LDczLDE0MCwyMjIsMjI5LDEzMCwxODksMzcsOTQsMjEwLDE1MiwxNDEsOTUsMjQ4LDIyMyw0NCwxNzcsMTI5LDcwLDQzLDkyLDIxNCw4MiwxNjAsMTgwLDEzNCwyMTcsMjEwLDE5NCw0MywyMTcsMTYzLDE0MiwxODIsMTQzLDE0MSwxNzEsMTgzLDIyNV19XX0",
    "policy_ids": []
}
```

and the protobuf definition file declared in [API section](#api)

```shell
# Use the following cmdline to install grpcurl
# go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

REQ=$(cat <path-to-request>)
grpcurl \
  -plaintext \
  -import-path <path-to-parent-directory-of-proto-file> \
  -proto <path-to-proto-file> \
  -d @ 127.0.0.1:50004 attestation.AttestationService/AttestationEvaluate <<EOF
$REQ
EOF
```

Then you can see a token looking like the following
```json
{
  "attestationToken": "eyJhbGciOiJSUzM4NCIsInR5cCI6IkpXVCJ9.eyJjdXN0b21pemVkX2NsYWltcyI6eyJ0ZXN0X2tleSI6InRlc3RfdmFsdWUifSwiZXZhbHVhdGlvbi1yZXBvcnRzIjpbeyJldmFsdWF0aW9uLXJlc3VsdCI6IntcImFsbG93XCI6dHJ1ZX0iLCJwb2xpY3ktaGFzaCI6ImMwZTc5Mjk2NzFmYjY3ODAzODdmNTQ3NjBkODRkNjVkMmNlOTYwOTNkZmIzM2VmZGEyMWY1ZWIwNWFmY2RhNzdiYmE0NDRjMDJjZDE3N2IyM2E1ZDM1MDcxNjcyNjE1NyIsInBvbGljeS1pZCI6ImRlZmF1bHQifV0sImV4cCI6MTcwMTY3Mjk2NSwiaXNzIjoiQ29Dby1BdHRlc3RhdGlvbi1TZXJ2aWNlIiwiandrIjp7ImFsZyI6IlJTMzg0IiwiZSI6IkFRQUIiLCJrdHkiOiJSU0EiLCJuIjoiMGhGUHdHNmdTSmJKV1NaTFR6SzRfVThHSWo5WVdYc3FZbDFjbUhveHlhQ05oQ3JCYVVUcW5KdHlPRVpMLVBDcWJBS2VXNGFCWnY1M3Zycm13OU41S2lHMHNOTVJOMUc1V2V2RTFNSEEzQU1qNlNlSWtwT1hzT01DNzJBNUZrZFIzRG1hM3dMaW5tZUVHYk9xZE5rN2IzMHdtWkRhVG13QTJJSjdnNVhPZk8zNTl6YWFLaDFRZDdPUXRkT2RfaV8tQlEzQlpEYnZ4R1ctWmRsdHVwWXBjRVQwWUZLSlE1NTdPSGtsOGMxT3BVdFc5ODlEQjM3d1BGTlRxM25oU3ZveDBwYWdDd3FwZ3JCYXVVUDBlOGlkX1VhSGFVZWlPd2tXc2UxdkdYQW55cFZqUlhhdERhS2dzbzZ5QjdGQ3pMUmRwM3JzWW1kd1lMaTdtMms3TkNPaE9RIn0sIm5iZiI6MTcwMTY3MjY2NSwidGNiLXN0YXR1cyI6eyJzZ3guYm9keS5hdHRyaWJ1dGVzLmZsYWdzIjoiMDcwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LmF0dHJpYnV0ZXMueGZybSI6ImU3MDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5jb25maWdfaWQiOiIwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LmNvbmZpZ19zdm4iOiIwMDAwIiwic2d4LmJvZHkuY3B1X3N2biI6IjA2MDYwYzBjZmZmZjAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkuaXN2X2V4dF9wcm9kX2lkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5pc3ZfZmFtaWx5X2lkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5pc3ZfcHJvZF9pZCI6IjAwMDAiLCJzZ3guYm9keS5pc3Zfc3ZuIjoiMDAwMCIsInNneC5ib2R5Lm1pc2Nfc2VsZWN0IjoiMDEwMDAwMDAiLCJzZ3guYm9keS5tcl9lbmNsYXZlIjoiOGYxNzNlNDYxM2ZmMDVjNTJhYWYwNDE2MmQyMzRlZGFlOGM5OTc3ZWFlNDdlYjIyOTlhZTE2YTU1MzAxMWM2OCIsInNneC5ib2R5Lm1yX3NpZ25lciI6IjgzZDcxOWU3N2RlYWNhMTQ3MGY2YmFmNjJhNGQ3NzQzMDNjODk5ZGI2OTAyMGY5YzcwZWUxZGZjMDhjN2NlOWUiLCJzZ3guYm9keS5yZXBvcnRfZGF0YSI6Ijc0NjU3Mzc0MDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkucmVzZXJ2ZWQxIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkucmVzZXJ2ZWQyIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LnJlc2VydmVkMyI6IjAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5yZXNlcnZlZDQiOiIwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guaGVhZGVyLmF0dF9rZXlfZGF0YV8wIjoiMDAwMDAwMDAiLCJzZ3guaGVhZGVyLmF0dF9rZXlfdHlwZSI6IjAyMDAiLCJzZ3guaGVhZGVyLnBjZV9zdm4iOiIwZDAwIiwic2d4LmhlYWRlci5xZV9zdm4iOiIwODAwIiwic2d4LmhlYWRlci51c2VyX2RhdGEiOiJkY2NkZTliMzFjZTg4NjA1NDgxNzNiYjRhMmE1N2ExNjAwMDAwMDAwIiwic2d4LmhlYWRlci52ZW5kb3JfaWQiOiI5MzlhNzIzM2Y3OWM0Y2E5OTQwYTBkYjM5NTdmMDYwNyIsInNneC5oZWFkZXIudmVyc2lvbiI6IjAzMDAifX0.SG7TUxm0E3yZs7rozijScMJIZTY8WVPZN3Yxu2CsW8HFE6lDLymdTzc1XTVrYb97PpGc6oCLwuLax786XHLN250SY_IW5GmR5WKRcYSGSQtOnYfsY7AMX3hvpV3rHGjP0QWZo_ezUp9yIbnJNwSprmFTzcNZkr2YNr1KmwWU-LhGSVCyviQwgtnqnhmQGwH-nHCcmgk0F3su_hdoFXImCggSHStXECAJ0cNjpAuTCsSQvrB4g3lM-dMii-D6a58uB_TGuOVf8Yqj9Gi6PrdxvIJZc1LSDDgo9uYuavNzunU3S3TkA2ZLDK4HIB0zDWfOnS2ZTrjyRdu5ZkzoGVK_YQ"
}
```

### API

The API of gRPC CoCo-AS is defined by the following proto file
```proto
syntax = "proto3";

package attestation;

message AttestationRequest {
    // TEE enum. Specify the evidence type
    string tee = 1;

    // Base64 encoded evidence. The alphabet is URL_SAFE_NO_PAD.
    // defined in https://datatracker.ietf.org/doc/html/rfc4648#section-5
    string evidence = 2;

    // Runtime Data used to check the binding relationship with report data in
    // Evidence
    oneof runtime_data {
        // Base64 encoded runtime data slice. The whole string will be base64
        // decoded. The result one will then be accumulated into a digest which
        // is used as the expected runtime data to check against the one inside
        // evidence.
        //
        // The alphabet is URL_SAFE_NO_PAD.
        // defined in https://datatracker.ietf.org/doc/html/rfc4648#section-5
        string raw_runtime_data = 3;

        // Runtime data in a JSON map. CoCoAS will rearrange each layer of the
        // data JSON object in dictionary order by key, then serialize and output
        // it into a compact string, and perform hash calculation on the whole
        // to check against the one inside evidence.
        //
        // After the verification, the structured runtime data field will be included
        // inside the token claims.
        string structured_runtime_data = 4;
    }

    // Init Data used to check the binding relationship with init data in
    // Evidence
    oneof init_data {
        // Base64 encoded init data slice. The whole string will be base64
        // decoded. The result one will then be accumulated into a digest which
        // is used as the expected init data to check against the one inside
        // evidence.
        //
        // The alphabet is URL_SAFE_NO_PAD.
        // defined in https://datatracker.ietf.org/doc/html/rfc4648#section-5
        string raw_init_data = 5;

        // Init data in a JSON map. CoCoAS will rearrange each layer of the
        // data JSON object in dictionary order by key, then serialize and output
        // it into a compact string, and perform hash calculation on the whole
        // to check against the one inside evidence.
        // 
        // After the verification, the structured init data field will be included
        // inside the token claims.
        string structured_init_data = 6;
    }

    // Hash algorithm used to calculate runtime data. Currently can be "sha256",
    // "sha384" or "sha512". If not specified, "sha384" will be selected.
    string runtime_data_hash_algorithm = 7;

    // Hash algorithm used to calculate init data. Currently can be "sha256",
    // "sha384" or "sha512". If not specified, "sha384" will be selected.
    string init_data_hash_algorithm = 8;

    // List of IDs of the policy used to check evidence. If not provided,
    // a "default" one will be used.
    repeated string policy_ids = 9;
}

message AttestationResponse {
    string attestation_token = 1;
}

message SetPolicyRequest {
    string policy_id = 1;
    string policy = 2;
}
message SetPolicyResponse {}

message ChallengeRequest {
    // ChallengeRequest uses HashMap to pass variables like:
    // tee, tee_params etc
    map<string, string> inner = 1;
}
message ChallengeResponse {
    string attestation_challenge = 1;
}

service AttestationService {
    rpc AttestationEvaluate(AttestationRequest) returns (AttestationResponse) {};
    rpc SetAttestationPolicy(SetPolicyRequest) returns (SetPolicyResponse) {};
    rpc GetAttestationChallenge(ChallengeRequest) returns (ChallengeResponse) {};
    // Get the GetPolicyRequest.user and GetPolicyRequest.tee specified Policy(.rego)
}
```

## RESTful CoCoAS

### Quick Start and Test

Users can use a [community version of restful CoCoAS image](https://github.com/confidential-containers/trustee/pkgs/container/staged-images%2Fcoco-as-restful) to verify attestation reports.

```shell
# run restful CoCoAS server locally
docker run -d \
  -v <path-to-attestation-service>/docs/sgx_default_qcnl.conf:/etc/sgx_default_qcnl.conf \ # this qcnl config is used when verifying SGX/TDX quotes
  -p 8080:8080 \
  ghcr.io/confidential-containers/staged-images/coco-as-restful:latest \
  restful-as
```

The `sgx_default_qcnl.conf` configures the PCS (Provisioning Certification Service) / PCCS (Provisioning Certification Cache Service) of Intel platforms. If you want to verify SGX/TDX quotes, please refer to the [Intel Platform Configuration](#intel-platform-configuration) section.

We can use `curl` to test the RESTful API using [the same request body](#quick-start-and-test) declared in the gRPC part.

```shell
cd <path-to-attestation-service>
curl -k -X POST http://127.0.0.1:8080/attestation \
     -i \
     -H 'Content-Type: application/json' \
     -d @tests/coco-as/request.json
```

Then, a token will be retrieved as HTTP response body like
```plaintext
eyJhbGciOiJSUzM4NCIsInR5cCI6IkpXVCJ9.eyJjdXN0b21pemVkX2NsYWltcyI6eyJ0ZXN0X2tleSI6InRlc3RfdmFsdWUifSwiZXZhbHVhdGlvbi1yZXBvcnRzIjpbeyJldmFsdWF0aW9uLXJlc3VsdCI6IntcImFsbG93XCI6dHJ1ZX0iLCJwb2xpY3ktaGFzaCI6ImMwZTc5Mjk2NzFmYjY3ODAzODdmNTQ3NjBkODRkNjVkMmNlOTYwOTNkZmIzM2VmZGEyMWY1ZWIwNWFmY2RhNzdiYmE0NDRjMDJjZDE3N2IyM2E1ZDM1MDcxNjcyNjE1NyIsInBvbGljeS1pZCI6ImRlZmF1bHQifV0sImV4cCI6MTcwMTY3Mjk2NSwiaXNzIjoiQ29Dby1BdHRlc3RhdGlvbi1TZXJ2aWNlIiwiandrIjp7ImFsZyI6IlJTMzg0IiwiZSI6IkFRQUIiLCJrdHkiOiJSU0EiLCJuIjoiMGhGUHdHNmdTSmJKV1NaTFR6SzRfVThHSWo5WVdYc3FZbDFjbUhveHlhQ05oQ3JCYVVUcW5KdHlPRVpMLVBDcWJBS2VXNGFCWnY1M3Zycm13OU41S2lHMHNOTVJOMUc1V2V2RTFNSEEzQU1qNlNlSWtwT1hzT01DNzJBNUZrZFIzRG1hM3dMaW5tZUVHYk9xZE5rN2IzMHdtWkRhVG13QTJJSjdnNVhPZk8zNTl6YWFLaDFRZDdPUXRkT2RfaV8tQlEzQlpEYnZ4R1ctWmRsdHVwWXBjRVQwWUZLSlE1NTdPSGtsOGMxT3BVdFc5ODlEQjM3d1BGTlRxM25oU3ZveDBwYWdDd3FwZ3JCYXVVUDBlOGlkX1VhSGFVZWlPd2tXc2UxdkdYQW55cFZqUlhhdERhS2dzbzZ5QjdGQ3pMUmRwM3JzWW1kd1lMaTdtMms3TkNPaE9RIn0sIm5iZiI6MTcwMTY3MjY2NSwidGNiLXN0YXR1cyI6eyJzZ3guYm9keS5hdHRyaWJ1dGVzLmZsYWdzIjoiMDcwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LmF0dHJpYnV0ZXMueGZybSI6ImU3MDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5jb25maWdfaWQiOiIwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LmNvbmZpZ19zdm4iOiIwMDAwIiwic2d4LmJvZHkuY3B1X3N2biI6IjA2MDYwYzBjZmZmZjAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkuaXN2X2V4dF9wcm9kX2lkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5pc3ZfZmFtaWx5X2lkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5pc3ZfcHJvZF9pZCI6IjAwMDAiLCJzZ3guYm9keS5pc3Zfc3ZuIjoiMDAwMCIsInNneC5ib2R5Lm1pc2Nfc2VsZWN0IjoiMDEwMDAwMDAiLCJzZ3guYm9keS5tcl9lbmNsYXZlIjoiOGYxNzNlNDYxM2ZmMDVjNTJhYWYwNDE2MmQyMzRlZGFlOGM5OTc3ZWFlNDdlYjIyOTlhZTE2YTU1MzAxMWM2OCIsInNneC5ib2R5Lm1yX3NpZ25lciI6IjgzZDcxOWU3N2RlYWNhMTQ3MGY2YmFmNjJhNGQ3NzQzMDNjODk5ZGI2OTAyMGY5YzcwZWUxZGZjMDhjN2NlOWUiLCJzZ3guYm9keS5yZXBvcnRfZGF0YSI6Ijc0NjU3Mzc0MDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkucmVzZXJ2ZWQxIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkucmVzZXJ2ZWQyIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LnJlc2VydmVkMyI6IjAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5yZXNlcnZlZDQiOiIwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guaGVhZGVyLmF0dF9rZXlfZGF0YV8wIjoiMDAwMDAwMDAiLCJzZ3guaGVhZGVyLmF0dF9rZXlfdHlwZSI6IjAyMDAiLCJzZ3guaGVhZGVyLnBjZV9zdm4iOiIwZDAwIiwic2d4LmhlYWRlci5xZV9zdm4iOiIwODAwIiwic2d4LmhlYWRlci51c2VyX2RhdGEiOiJkY2NkZTliMzFjZTg4NjA1NDgxNzNiYjRhMmE1N2ExNjAwMDAwMDAwIiwic2d4LmhlYWRlci52ZW5kb3JfaWQiOiI5MzlhNzIzM2Y3OWM0Y2E5OTQwYTBkYjM5NTdmMDYwNyIsInNneC5oZWFkZXIudmVyc2lvbiI6IjAzMDAifX0.SG7TUxm0E3yZs7rozijScMJIZTY8WVPZN3Yxu2CsW8HFE6lDLymdTzc1XTVrYb97PpGc6oCLwuLax786XHLN250SY_IW5GmR5WKRcYSGSQtOnYfsY7AMX3hvpV3rHGjP0QWZo_ezUp9yIbnJNwSprmFTzcNZkr2YNr1KmwWU-LhGSVCyviQwgtnqnhmQGwH-nHCcmgk0F3su_hdoFXImCggSHStXECAJ0cNjpAuTCsSQvrB4g3lM-dMii-D6a58uB_TGuOVf8Yqj9Gi6PrdxvIJZc1LSDDgo9uYuavNzunU3S3TkA2ZLDK4HIB0zDWfOnS2ZTrjyRdu5ZkzoGVK_YQ
```

### API

RESTful CoCo-AS's endpoints are as following:

- `/attestation`: receives evidence verification request. The request POST payload is like
```json
{
    "tee": "sgx", // tee type.
    "evidence": "YWFhCg==...", // base64 encoded evidence in URL SAFE NO PAD,
    "runtime_data": {           // `runtime_data` is optional. If given, the runtime data binding will
                                // be checked.
                                // The field `raw` and `structured` are exclusive.
        "raw": "YWFhCg==...",   // Base64 encoded runtime data slice. The whole string will be base64
                                // decoded. The result one will then be accumulated into a digest which
                                // is used as the expected runtime data to check against the one inside
                                // evidence.
                                //
                                // The alphabet is URL_SAFE_NO_PAD.
                                // defined in https://datatracker.ietf.org/doc/html/rfc4648#section-5
                                
        "structured": {}        // Runtime data in a JSON map. CoCoAS will rearrange each layer of the
                                // data JSON object in dictionary order by key, then serialize and output
                                // it into a compact string, and perform hash calculation on the whole
                                // to check against the one inside evidence. The hash algorithm is defined
                                // by `runtime_data_hash_algorithm`.
                                //
                                // After the verification, the structured runtime data field will be included
                                // inside the token claims.
    }, 
    "init_data": {              // `init_data` is optional. If given, the init data binding will
                                // be checked.
                                // The field `raw` and `structured` are exclusive.
        "raw": "YWFhCg==...",   // Base64 encoded init data slice. The whole string will be base64
                                // decoded. The result one will then be accumulated into a digest which
                                // is used as the expected init data to check against the one inside
                                // evidence. The hash algorithm is defined by `init_data_hash_algorithm`.
                                //
                                // The alphabet is URL_SAFE_NO_PAD.
                                // defined in https://datatracker.ietf.org/doc/html/rfc4648#section-5
                                
        "structured": {}        // Init data in a JSON map. CoCoAS will rearrange each layer of the
                                // data JSON object in dictionary order by key, then serialize and output
                                // it into a compact string, and perform hash calculation on the whole
                                // to check against the one inside evidence.
                                //
                                // After the verification, the structured init data field will be included
                                // inside the token claims.
    }, 
    "runtime_data_hash_algorithm": "sha384",// Hash algorithm used to calculate runtime data. Currently can be 
                                            // "sha256", "sha384" or "sha512". If not specified, "sha384" will be selected.
    "init_data_hash_algorithm": "sha384",   // Hash algorithm used to calculate init data. Currently can be 
                                            // "sha256", "sha384" or "sha512". If not specified, "sha384" will be selected.
    "policy_ids": ["default", "policy-1"]           // List of IDs of the policy used to check evidence. If
                                                    // not provided, a "default" one will be used.
}
```
- `/policy`: receives policy setting request. The request POST payload is like
```json
{
    "type": "rego",         // policy type
    "policy_id": "yyyyy",   // raw string of policy id
    "policy": "xxxxx"       // base64 encoded policy content
}
```

## Launch Configuration

For both gRPC and RESTful version CoCoAS, you can use the `-c` parameter to specify the path of the configuration file.

The configuration file is a JSON file.

The following properties can be set globally, i.e. not under any configuration
section:

| Property                   | Type                        | Description                                         | Required | Default |
|----------------------------|-----------------------------|-----------------------------------------------------|----------|---------|
| `work_dir`                 | String                      | The location for Attestation Service to store data. | False      | Firstly try to read from ENV `AS_WORK_DIR`. If not any, use `/opt/confidential-containers/attestation-service`       |
| `rvps_config`              | [RVPSConfiguration][2]      | RVPS configuration                                  | False      | -       |
| `attestation_token_broker` | [AttestationTokeBroker][1]  | Attestation result token configuration.             | False      | -       |

[1]: #attestationtokenbroker
[2]: #rvps-configuration

#### AttestationTokenBroker

| Property       | Type                    | Description                                          | Required | Default |
|----------------|-------------------------|------------------------------------------------------|----------|---------|
| `type`         | String                  | Type of token to issue (`Ear` or `Simple`)               | No       | `Ear`   |

When `type` field is set to `Ear`, the following extra properties can be set:

| Property       | Type                    | Description                                          | Required | Default |
|----------------|-------------------------|------------------------------------------------------|----------|---------|
| `duration_min` | Integer                 | Duration of the attestation result token in minutes. | No       | `5`     |
| `issuer_name`  | String                  | Issuer name of the attestation result token.         | No       |`CoCo-Attestation-Service`|
| `developer_name`  | String               | The developer name to be used as part of the Verifier ID in the EAR | No       |`https://confidentialcontainers.org`|
| `build_name`  | String                  | The build name to be used as part of the Verifier ID in the EAR         | No       | Automatically generated from Cargo package and AS version|
| `profile_name`  | String                  | The Profile that describes the EAR token         | No       |tag:github.com,2024:confidential-containers/Trustee`|
| `policy_dir`  | String                  | The path to the work directory that contains policies to provision the tokens.        | No       |`/opt/confidential-containers/attestation-service/token/ear/policies`|
| `signer`       | [TokenSignerConfig][1]  | Signing material of the attestation result token.    | No       | None       |

[1]: #tokensignerconfig

When `type` field is set to `Simple`, the following extra properties can be set:
| Property       | Type                    | Description                                          | Required | Default |
|----------------|-------------------------|------------------------------------------------------|----------|---------|
| `duration_min` | Integer                 | Duration of the attestation result token in minutes. | No       | `5`     |
| `issuer_name`  | String                  | Issuer name of the attestation result token.         | No       |`CoCo-Attestation-Service`|
| `policy_dir`  | String                  | The path to the work directory that contains policies to provision the tokens.        | No       |`/opt/confidential-containers/attestation-service/token//simple/policies`|
| `signer`       | [TokenSignerConfig][1]  | Signing material of the attestation result token.    | No       | None       |

[1]: #tokensignerconfig

##### TokenSignerConfig

This section is **optional**. When omitted, a new RSA key pair is generated and used.

| Property       | Type    | Description                                              | Required | Default |
|----------------|---------|----------------------------------------------------------|----------|---------|
| `key_path`     | String  | RSA Key Pair file (PEM format) path.                     | Yes      | -       |
| `cert_url`     | String  | RSA Public Key certificate chain (PEM format) URL.       | No       | -       |
| `cert_path`    | String  | RSA Public Key certificate chain (PEM format) file path. | No       | -       |

#### RVPS Configuration

| Property       | Type                    | Description                                          | Required | Default |
|----------------|-------------------------|------------------------------------------------------|----------|---------|
| `type`         | String                  | It can be either `BuiltIn` (Built-In RVPS) or `GrpcRemote` (connect to a remote gRPC RVPS) | No       | `BuiltIn` |

##### BuiltIn RVPS

If `type` is set to `BuiltIn`, the following extra properties can be set

| Property       | Type                    | Description                                                           | Required | Default  |
|----------------|-------------------------|-----------------------------------------------------------------------|----------|----------|
| `store_type`   | String                  | The underlying storage type of RVPS. (`LocalFs` or `LocalJson`)       | No       | `LocalFs`|
| `store_config` | JSON Map                | The optional configurations to the underlying storage.                | No       | Null     |

Different `store_type` will have different `store_config` items.

For `LocalFs`, the following properties can be set

| Property       | Type                    | Description                                              | Required | Default  |
|----------------|-------------------------|----------------------------------------------------------|----------|----------|
| `file_path`    | String                  | The path to the directory storing reference values       | No       | `/opt/confidential-containers/attestation-service/reference_values`|

For `LocalJson`, the following properties can be set

| Property       | Type                    | Description                                              | Required | Default  |
|----------------|-------------------------|----------------------------------------------------------|----------|----------|
| `file_path`    | String                  | The path to the file that storing reference values       | No       | `/opt/confidential-containers/attestation-service/reference_values.json`|

##### Remote RVPS

If `type` is set to `GrpcRemote`, the following extra properties can be set

| Property       | Type                    | Description                             | Required | Default          |
|----------------|-------------------------|-----------------------------------------|----------|------------------|
| `address`      | String                  | Remote address of the RVPS server       | No       | `127.0.0.1:50003`|


#### Configuration Examples

Running with a built-in RVPS:

```json
{
    "work_dir": "/var/lib/attestation-service/",
    "policy_engine": "opa",
    "rvps_config": {
        "type": "BuiltIn",
        "store_type": "LocalFs",
        "store_config": {
            "file_path": "/var/lib/attestation-service/reference-values"
        }
    },
    "attestation_token_broker": {
        "type": "Ear",
        "duration_min": 5
    }
}
```

Running with a remote RVPS:

```json
{
    "work_dir": "/var/lib/attestation-service/",
    "policy_engine": "opa",
    "rvps_config": {
        "type": "GrpcRemote",
        "address": "127.0.0.1:50003"
    },
    "attestation_token_broker": {
	"type": "Ear",
        "duration_min": 5
    }
}
```

Configurations for token signer

```json
{
    "work_dir": "/var/lib/attestation-service/",
    "policy_engine": "opa",
    "rvps_config": {
        "type": "GrpcRemote",
        "address": "127.0.0.1:50003"
    },
    "attestation_token_broker": {
	"type": "Ear",
        "duration_min": 5,
        "issuer_name": "some-body",
        "signer": {
            "key_path": "/etc/coco-as/signer.key",
            "cert_url": "https://example.io/coco-as-certchain",
            "cert_path": "/etc/coco-as/signer.pub"
        }
    }
}
```


## Backups

### Intel Platform Configuration

A workable file which will directly connect to [Intel's PCS](https://api.portal.trustedservices.intel.com/provisioning-certification) without caching is given.

```json
{"collateral_service": "https://api.trustedservices.intel.com/sgx/certification/v4/"}
```

This can be used for test. Users are expected to set the file to connect to another available PCCS which keeps cache.
PCCS are usually supported by cloud providers, you can find the steps to configure `/etc/sgx_default_qcnl.conf` for
- Aliyun (Alibaba Cloud): [Build an SGX confidential computing environment](https://www.alibabacloud.com/help/en/ecs/user-guide/build-an-sgx-encrypted-computing-environment)
- Azure: [Trusted Hardware Identity Management](https://learn.microsoft.com/en-us/azure/security/fundamentals/trusted-hardware-identity-management)
- IBM Cloud: [Attestation with Intel SGX and Data Center Attestation Primitives (DCAP) for Virtual Servers for VPC](https://cloud.ibm.com/docs/vpc?topic=vpc-about-attestation-sgx-dcap-vpc)
Or you can [set-up a PCCS yourself](https://download.01.org/intel-sgx/sgx-dcap/1.9/windows/docs/Intel_SGX_DCAP_Windows_SW_Installation_Guide.pdf).