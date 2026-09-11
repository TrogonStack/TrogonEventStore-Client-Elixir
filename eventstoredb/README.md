# TrogonEventStore test configuration

Modified by Straw Hat, LLC in 2026.

This directory contains TrogonEventStore configuration used for integration
testing.

## Certificates

Certificates may be generated on-demand with the gen-certs.sh script:

```
./eventstoredb/gen-certs.sh
```

This writes a throwaway CA and node certificate into `eventstoredb/certs/`
(git-ignored) with a 10 year expiry.

You can inspect a generated certificate with `openssl`:

```
openssl x509 -in eventstoredb/certs/node.crt -text
```

## Server image

The Docker Compose stack defaults to
`ghcr.io/trogonstack/trogoneventstore:ci`. Set `TROGON_EVENTSTORE_IMAGE` to
verify the client against another fully qualified image.
