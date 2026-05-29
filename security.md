# LegacyTel Security

LegacyTel transports sensitive operational and security audit data from legacy platforms to modern SIEM and observability systems. This document summarizes the security controls and operational practices for deploying LegacyTel safely.

## Security Scope

LegacyTel is designed to handle audit and operational records from:

- IBM z/OS SMF records, including RACF security events.
- IBM i / AS/400 QAUDJRN audit journal entries.
- HPE NonStop / Tandem EMS events.

These records can contain sensitive metadata about users, authentication, authorization, system configuration, and administrative activity.

## Data in Transit

LegacyTel receivers support Transport Layer Security (TLS) for encrypted TCP log streams.

- Enable TLS per receiver with `tls_enabled: true`.
- Configure the server certificate with `cert_file`.
- Configure the server private key with `key_file`.
- LegacyTel enforces TLS 1.2 or newer for secure receiver connections.

Example receiver settings:

```yaml
receivers:
  zos_smf:
    enabled: true
    bind_address: "0.0.0.0"
    port: 5080
    tls_enabled: true
    cert_file: "/etc/legacytel/certs/server.crt"
    key_file: "/etc/legacytel/certs/server.key"
```

## Mutual TLS

For production and zero-trust environments, enable mutual TLS (mTLS) so only trusted legacy source systems can connect.

- Set `client_ca_file` to the trusted certificate authority used to sign client certificates.
- LegacyTel requires and verifies client certificates when `client_ca_file` is configured.
- Source systems must present a valid client certificate signed by the configured CA.

Example mTLS receiver settings:

```yaml
receivers:
  as400_qaudjrn:
    enabled: true
    bind_address: "0.0.0.0"
    port: 5081
    tls_enabled: true
    cert_file: "/etc/legacytel/certs/server.crt"
    key_file: "/etc/legacytel/certs/server.key"
    client_ca_file: "/etc/legacytel/certs/root_ca.crt"
```

## Certificate Generation

The repository includes `generate_certs.sh`, which creates:

- `root_ca.crt` and `root_ca.key`
- `server.crt` and `server.key`
- `zos_mainframe_client.crt` and `zos_mainframe_client.key`
- `as400_iseries_client.crt` and `as400_iseries_client.key`
- `tandem_nonstop_client.crt` and `tandem_nonstop_client.key`

Run the utility from the repository root:

```bash
chmod +x generate_certs.sh
./generate_certs.sh
```

Store private keys securely and do not commit generated certificates or keys to source control.

## File Permissions

Use restrictive permissions for deployed configuration and key material.

```bash
sudo chmod 600 /etc/legacytel/certs/server.key
sudo chmod 644 /etc/legacytel/certs/server.crt
sudo chmod 644 /etc/legacytel/certs/root_ca.crt
```

Limit access to `/etc/legacytel`, `/etc/legacytel/certs`, and any generated client private keys to trusted administrators and service accounts only.

## Network Exposure

Production deployments should restrict network access to LegacyTel listener ports:

- `5080` for z/OS SMF streams.
- `5081` for IBM i / AS/400 QAUDJRN streams.
- `5082` for HPE NonStop / Tandem EMS streams.
- `8080` for the embedded dashboard, when enabled.

Use firewalls, network segmentation, and allowlists so only approved source systems and operators can reach these ports.

## Dashboard Access

The embedded dashboard is useful for real-time observability, but it can expose operational telemetry. In production:

- Bind the dashboard to a restricted interface when possible.
- Place it behind an authenticated reverse proxy if remote access is required.
- Disable it with `enable_dashboard: false` when not needed.

## Exporter Security

LegacyTel can forward events to OTLP/HTTP, syslog, and Splunk HEC endpoints. Protect exporter traffic by:

- Using HTTPS endpoints for Splunk HEC and OTLP/HTTP where supported.
- Avoiding `insecure_skip_verify: true` in production.
- Treating exporter tokens and authorization headers as secrets.
- Storing production tokens outside public source control.

## Vulnerability Reporting

If you discover a security issue in LegacyTel, do not disclose it publicly until it has been reviewed and remediated. Report the issue privately to the project maintainers with:

- A description of the vulnerability.
- Steps to reproduce or proof-of-concept details.
- Affected versions, configurations, or deployment modes.
- Suggested remediation, if available.

## Production Hardening Checklist

- [ ] Enable TLS for all receiver ports that carry audit data.
- [ ] Enable mTLS with `client_ca_file` for trusted source verification.
- [ ] Protect all private keys with restrictive file permissions.
- [ ] Restrict listener ports with firewall rules and network allowlists.
- [ ] Disable or protect the dashboard in production.
- [ ] Use secure exporter endpoints and validate upstream certificates.
- [ ] Keep generated credentials out of source control.
- [ ] Rotate certificates and tokens according to organizational policy.
