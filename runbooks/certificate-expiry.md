# Runbook: SSL Certificate Expiry

**Alert name:** `CertificateExpiringInDays`  
**Severity:** Warning (30d), Critical (7d), Page (2d)  
**Trigger:** ACM certificate expiry OR GKE Certificate Map renewal failure  
**Team:** Platform SRE

---

## AWS ACM Certificates

ACM certificates auto-renew if:
1. DNS validation records are present in Route53
2. Certificate is attached to an active ALB/CloudFront distribution

```bash
# List all ACM certificates and their expiry
aws acm list-certificates --region ap-southeast-1 \
  --query 'CertificateSummaryList[*].{Domain:DomainName,ARN:CertificateArn}' | \
  jq -r '.[] | .Domain + " " + .ARN' | \
  while read domain arn; do
    expiry=$(aws acm describe-certificate --region ap-southeast-1 \
      --certificate-arn "$arn" \
      --query 'Certificate.NotAfter' --output text)
    echo "$domain  expires: $expiry"
  done

# Force renewal (for stuck certificates)
aws acm renew-certificate --certificate-arn $CERT_ARN --region ap-southeast-1
```

### Common failure: CNAME validation record deleted from Route53

```bash
# 1. Get required CNAME record
aws acm describe-certificate \
  --certificate-arn $CERT_ARN \
  --region ap-southeast-1 \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord'

# 2. Add it back to Route53
# Output example: Name: _abc.domain.com, Value: _xyz.acm-validations.aws
# Add as CNAME record in the appropriate hosted zone TF file
```

---

## GKE Certificate Map (Google-managed)

GKE uses Certificate Manager with auto-renewal via DNS challenge.

```bash
# Check certificate status
gcloud certificate-manager certificates list --project prj-acme-prd

# Describe a specific cert for renewal status
gcloud certificate-manager certificates describe $CERT_NAME --project prj-acme-prd

# Check DNS authorization
gcloud certificate-manager dns-authorizations list --project prj-acme-prd
```

### Common failure: DNS authorization CNAME missing

```bash
# 1. Get required CNAME
gcloud certificate-manager dns-authorizations describe $DNS_AUTH_NAME \
  --project prj-acme-prd \
  --format="json" | jq '.dnsResourceRecord'

# 2. Add the CNAME to the Route53 hosted zone (same as above)
```

---

## Self-Signed / Manual Certificates (ClickHouse, internal services)

```bash
# Check expiry of a certificate file
openssl x509 -in /path/to/cert.pem -noout -dates

# Renew with Let's Encrypt (if applicable)
certbot renew --cert-name $DOMAIN --dry-run
certbot renew --cert-name $DOMAIN

# Restart service after cert renewal
sudo systemctl restart nginx  # or clickhouse-server, etc.
```

---

## Monitoring Certificate Expiry

Add to Prometheus alerting rules (see gke-observability-stack):

```yaml
- alert: CertificateExpiryWarning
  expr: ssl_certificate_expiry_seconds < 30 * 24 * 3600
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Certificate for {{ $labels.domain }} expires in < 30 days"

- alert: CertificateExpiryCritical
  expr: ssl_certificate_expiry_seconds < 7 * 24 * 3600
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: "Certificate for {{ $labels.domain }} expires in < 7 days — URGENT"
```

---

## Escalation

- **7d**: Warning alert fires → SRE to investigate and confirm auto-renewal is working
- **2d**: Page on-call → manually trigger renewal
- **Expired**: P1 incident — certificates don't auto-renew after expiry, requires manual replacement
