# Security & Compliance

Protecting data with layered security and meeting regulatory requirements.

## 📚 Core Topics

1. **Access Control** — Least privilege, RBAC
2. **Encryption** — At-rest and in-transit
3. **Audit Logging** — Compliance and forensics
4. **Network Security** — Firewall, VPN, TLS
5. **GDPR Compliance** — PII handling and deletion
6. **PCI-DSS** — Payment data security
7. **SOC 2** — Service organization controls
8. **Secret Management** — Key rotation, vault

## 🔐 Security Layers

```
Layer 5: AUDIT & MONITORING
  - DDL changes logged
  - Query audit on sensitive tables
  - SIEM integration

Layer 4: ENCRYPTION
  - At-rest (AES-256)
  - In-transit (TLS 1.2+)
  - Key management (KMS/HSM)

Layer 3: AUTHORIZATION
  - Least privilege roles
  - Schema separation
  - Row-level security (RLS)

Layer 2: AUTHENTICATION
  - No shared passwords
  - IAM/AD integration
  - MFA for DBA access
  - Service accounts per app

Layer 1: NETWORK
  - Private subnet
  - Security groups
  - Firewall rules
```

## 🚀 Implementation

### Network Security

```sql
-- PostgreSQL network configuration
-- postgresql.conf:
listen_addresses = '10.0.1.0/24'  -- Only private network

-- pg_hba.conf:
# Only allow app servers
host    mydb    app_user    10.0.1.0/24    md5

# DBA access via jump host
host    mydb    dba_user    10.0.2.50/32   md5
```

### Authentication Setup

```sql
-- Create service role (not superuser)
CREATE ROLE myapp_reader WITH LOGIN PASSWORD 'strong_password';
CREATE ROLE myapp_writer WITH LOGIN PASSWORD 'strong_password';

-- Grant minimal permissions
GRANT CONNECT ON DATABASE mydb TO myapp_reader;
GRANT USAGE ON SCHEMA public TO myapp_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO myapp_reader;

-- Writer role gets INSERT/UPDATE/DELETE
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO myapp_writer;

-- Never use superuser in application!
```

### Encryption Setup

```sql
-- Check if encryption at rest is enabled
SHOW ssl;  -- Should be 'on'
SHOW ssl_cert_file;

-- PostgreSQL: pgcrypto extension
CREATE EXTENSION pgcrypto;

-- Encrypt sensitive column
ALTER TABLE users ADD COLUMN ssn_encrypted bytea;

-- Encrypt data
UPDATE users SET ssn_encrypted = pgp_sym_encrypt(ssn, 'encryption_key');

-- Query encrypted data (decrypt on retrieval)
SELECT pgp_sym_decrypt(ssn_encrypted, 'encryption_key') AS ssn
FROM users WHERE id = 123;
```

### Audit Logging

```sql
-- PostgreSQL: pgaudit extension
CREATE EXTENSION pgaudit;

-- Log all DDL changes
ALTER SYSTEM SET pgaudit.log = 'DDL';

-- Log SELECT on sensitive tables
ALTER SYSTEM SET pgaudit.log_statement = 'all';
ALTER SYSTEM SET pgaudit.role = 'audit_role';

-- Reload configuration
SELECT pg_reload_conf();

-- View audit logs
tail -f /var/log/postgresql/postgresql.log | grep AUDIT
```

## 📋 Compliance Frameworks

### GDPR (General Data Protection Regulation)

**Key requirements:**

```
1. Data Inventory
   - Know what PII you have
   - Where it's stored
   - Who has access

2. Right to Delete
   - Delete customer PII on request
   - Prove deletion happened
   - Account for backups/replicas

3. Encryption
   - Sensitive data encrypted
   - In transit (TLS) and at rest

4. Access Control
   - Only necessary people access PII
   - Vendor agreements (data processors)

5. Breach Notification
   - 72-hour notification if breach
   - Document breach investigation
```

**Implementation:**

```sql
-- Tag sensitive tables
COMMENT ON TABLE users IS 'GDPR: Contains PII';
COMMENT ON COLUMN users.ssn IS 'GDPR: Sensitive, encrypted';

-- Data retention policy
DELETE FROM deleted_users WHERE deleted_at < NOW() - INTERVAL '90 days';

-- Backup retention for compliance (keep 7 years)
-- = 365 * 7 = 2555 days minimum retention

-- Document data flows
-- - What gets logged?
-- - What gets replicated?
-- - What gets backed up?
```

### PCI-DSS (Payment Card Industry Data Security Standard)

**Key requirements:**

```
1. Firewall configuration
   - Restrict access to cardholder data
   - VPN for remote access

2. No hardcoded passwords
   - Rotate regularly
   - Store in vault

3. Encrypt cardholder data
   - At rest and in transit
   - Tokenize when possible

4. Change detection
   - Alert on unauthorized changes
   - Audit all database changes

5. Testing
   - Annual penetration testing
   - Quarterly vulnerability scanning

6. Access control
   - Unique user IDs
   - Restrict by job necessity

7. Logs and monitoring
   - All access logged
   - Logs protected from deletion
```

**Implementation:**

```sql
-- Never store full payment card data!
-- Use tokenization instead

-- Bad:
CREATE TABLE payments (
    id SERIAL,
    card_number VARCHAR(16),  -- ❌ PCI-DSS violation!
    amount DECIMAL(10,2)
);

-- Good:
CREATE TABLE payments (
    id SERIAL,
    card_token VARCHAR(32),   -- Tokenized, not the actual card
    amount DECIMAL(10,2)
);

-- Store actual card at payment processor only
```

## 🔑 Secret Management

### Vault Setup (HashiCorp Vault)

```bash
# Store database credentials securely
vault kv put secret/databases/mydb \
  username=myapp_user \
  password=GeneratedStrongPassword123!

# Rotate credentials
vault read -field=password secret/databases/mydb

# Application retrieves secret at runtime
# (Not stored in config files!)
```

### Key Rotation Schedule

```
Database Credentials: Every 90 days
TLS Certificates: Every 365 days
Encryption Keys: Every year
Backup Encryption Keys: Every 2 years (or per compliance)
```

## 🛡️ Common Security Mistakes

```
❌ Shared passwords (multiple people, multiple systems)
❌ Superuser for application connections
❌ Unencrypted connections over network
❌ PII in non-prod environments
❌ No audit logging
❌ Weak backup security
❌ Direct database access from app servers (use proxy)
❌ Same password for dev/staging/prod
❌ No secret rotation policy
❌ Storing passwords in code/config

✓ Unique credentials per application
✓ Least privilege roles
✓ TLS everywhere
✓ Data masking in non-prod
✓ Comprehensive audit logs
✓ Encrypted backups in separate account
✓ Connection pooler/proxy
✓ Environment-specific secrets
✓ Automated secret rotation
✓ Vault-managed credentials
```

## 📊 Security Audit Checklist

**Network:**

- [ ] Database in private subnet
- [ ] Firewall rules whitelist only known IPs
- [ ] VPN required for DBA access
- [ ] TLS 1.2+ for all connections
- [ ] No public IP on database

**Access Control:**

- [ ] No shared passwords
- [ ] Service accounts per application
- [ ] Minimal privilege per role
- [ ] Superuser access disabled for app
- [ ] MFA for DBA console access
- [ ] Access review quarterly

**Encryption:**

- [ ] Encryption at rest enabled
- [ ] Encryption keys in KMS/HSM
- [ ] Key rotation policy implemented
- [ ] In-transit encryption (TLS)
- [ ] Sensitive columns encrypted

**Audit & Logging:**

- [ ] DDL changes logged
- [ ] Failed login attempts logged
- [ ] Query audit on sensitive tables
- [ ] Logs shipped to SIEM
- [ ] Log retention policy set
- [ ] Logs protected from tampering

**Backup & Recovery:**

- [ ] Backups encrypted
- [ ] Backup access restricted
- [ ] Offsite copy in separate account
- [ ] Immutable storage (WORM)
- [ ] Restore tested regularly
- [ ] Encryption keys backed up securely

**Compliance:**

- [ ] Data inventory documented
- [ ] Privacy policy aligned with data practices
- [ ] Vendor agreements in place
- [ ] Breach incident response plan
- [ ] Regular security assessments
- [ ] Penetration testing completed

## 📚 Interview Questions

1. **Design secure database for fintech application handling payment data**
   - Tokenization (no full card numbers stored)
   - PCI-DSS compliance (encryption, audit, access control)
   - TLS for all connections
   - Separate vault for secrets
   - Quarterly penetration testing
   - Immutable audit logs

2. **How do you handle user deletion request (GDPR)?**
   - Find all data for that user
   - Ensure backups accounted for
   - Delete from primary and replicas
   - Verify deletion with checksums
   - Document deletion for compliance
   - Note: Some backups may retain data (legal hold)

3. **What's your approach to secret management?**
   - No hardcoded credentials
   - Vault-managed secrets
   - Automated rotation (90 days)
   - Different secrets per environment
   - Limited TTL for credentials
   - Access logging to vault

## 🔗 Related

- [Backup & Recovery](../02-backup-recovery/)
- [Monitoring](../08-monitoring/)
- [PostgreSQL Guide](../07-platform-guides/postgresql.md)

---

**Key Insight:** Security isn't a feature you add at the end—it's built in from the start. Make it easy for developers to do the right thing (good APIs, clear documentation), and hard to do the wrong thing (no credentials in code, no direct access).
