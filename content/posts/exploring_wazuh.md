---
author: ["Anass"]
title: "Exploring Wazuh: SIEM, XDR and Keycloak SSO"
date: "2024-10-01"
description: "A practical tour of Wazuh’s SIEM and XDR capabilities, with a Keycloak SAML integration for centralized authentication and role mapping."
tags: ["Wazuh", "SIEM", "XDR", "Keycloak", "SAML", "Detection Engineering"]
categories: ["Security"]
topics: ["Security"]
content_type: "systems-note"
ShowToc: true
---

Wazuh is an open-source security platform built around endpoint monitoring, log analysis, vulnerability detection, configuration assessment and response. A typical deployment combines **Wazuh agents**, a **Wazuh server**, the **Wazuh indexer** and the **Wazuh dashboard**.

The platform becomes more useful when it is connected to the identity system that already governs access to the rest of the security estate. In this article, Keycloak provides centralized authentication and role ownership for the Wazuh dashboard. The integration documented by Wazuh uses **SAML**, not OpenID Connect: Keycloak issues a signed SAML assertion, and Wazuh maps the returned backend role to its own permissions.

This is not a copy-paste production recipe. Paths and interface labels can vary with the Wazuh and Keycloak versions in use. Treat the configuration as a lab baseline, replace every placeholder, and validate the complete login and authorization flow before removing the local administrator path.

<!--more-->

## The short version

- Wazuh collects and analyses telemetry from endpoints, workloads and infrastructure.
- Its main components are the agent, server, indexer and dashboard.
- Keycloak can act as the SAML Identity Provider for the Wazuh dashboard.
- Wazuh’s documented Keycloak integration uses a SAML client named `wazuh-saml`.
- The important security boundary is not only authentication: Keycloak roles must be mapped to the right Wazuh and indexer permissions.
- Start with a read-only role, keep a protected break-glass account, and test logout, role changes and certificate rotation.

> **Lab safety:** perform the configuration in a disposable or isolated environment first. Use test users, test certificates and non-production logs. Never paste a real Keycloak client secret, SAML signing key, Wazuh password or metadata containing sensitive internal addresses into a public article or repository.

## At a glance

| Field | Details |
| --- | --- |
| Platform | Wazuh server, indexer and dashboard |
| Identity Provider | Keycloak |
| Federation protocol | SAML 2.0 |
| Authentication flow | Browser redirect to Keycloak, then signed assertion to Wazuh |
| Authorization input | Keycloak role exposed as the SAML `Roles` attribute |
| Administrator mapping | `wazuh-admins` → Wazuh `administrator` / indexer administrative role |
| Read-only mapping | `wazuh-readonly` → read-only Wazuh and indexer permissions |
| Main risk | Correct authentication with excessive or missing authorization |
| Operational control | Versioned configuration, backups and a tested local fallback |

## What Wazuh actually does

Wazuh is easier to reason about when its responsibilities are separated:

- **Agents** collect events and state from monitored endpoints. They can report operating-system logs, file-integrity changes, vulnerability information, inventory and security-configuration results.
- **The Wazuh server** receives agent data, applies decoders and rules, and exposes the Wazuh API for management and automation.
- **The Wazuh indexer** stores security events and provides search and analytics capabilities.
- **The Wazuh dashboard** is the operator interface for investigations, alerts, compliance views, agent management and platform administration.

The dashboard is therefore a privileged control plane, not only a reporting page. Its login must be connected to a reliable identity lifecycle, and the permission mapping must be reviewed as carefully as any other administrative interface.

## The Keycloak integration model

Wazuh’s current integration guide documents SAML with Keycloak. The request and response path is:

1. An operator opens the Wazuh dashboard.
2. The dashboard sends the browser to the Keycloak realm used for Wazuh.
3. The user authenticates in Keycloak, including any MFA or required authentication flow.
4. Keycloak returns a signed SAML assertion to Wazuh’s Assertion Consumer Service.
5. The Wazuh indexer validates the assertion, issuer, metadata and signature.
6. The `Roles` attribute is exposed as a backend role such as `wazuh-admins` or `wazuh-readonly`.
7. Wazuh maps that backend role to the permissions used by the dashboard and indexer.

The result is centralized authentication with application-specific authorization. Keycloak decides who the user is and which roles they receive; Wazuh still decides what those roles can do inside the security platform.

## Before starting

Prepare the following values and dependencies:

| Value | Example |
| --- | --- |
| Keycloak realm | `Wazuh` |
| Keycloak base URL | `https://sso.example.test` |
| Wazuh dashboard URL | `https://wazuh.example.test` |
| SAML client / SP entity ID | `wazuh-saml` |
| Administrator backend role | `wazuh-admins` |
| Read-only backend role | `wazuh-readonly` |
| SAML role attribute | `Roles` |
| Metadata directory | `/etc/wazuh-indexer/opensearch-security/` |

The dashboard and Keycloak URLs must be stable and resolvable from the operator’s browser and from the Wazuh components that validate the SAML exchange. Use HTTPS, synchronize clocks with NTP, and make sure the certificate chain is trusted before troubleshooting SAML.

Back up the Wazuh indexer security configuration and the dashboard configuration before changing them. Keep the local administrator account available until an administrator has completed a successful SSO login and authorization test.

## Screenshots from the integration flow

The following interface captures come from the official Wazuh Keycloak integration documentation. They show the relevant screens and values; they are not screenshots of this blog’s own deployment.

![Keycloak Create realm screen with the Wazuh realm name.](/images/wazuh-keycloak/keycloak-create-realm.png)

<small><em>Creating the dedicated `Wazuh` realm in Keycloak · Wazuh documentation · <a href="https://documentation.wazuh.com/current/user-manual/user-administration/single-sign-on/keycloak.html">source</a></em></small>

![Keycloak SAML client settings for the Wazuh dashboard.](/images/wazuh-keycloak/keycloak-client-settings.png)

<small><em>Configuring the `wazuh-saml` client and its dashboard URL · Wazuh documentation · <a href="https://documentation.wazuh.com/current/user-manual/user-administration/single-sign-on/keycloak.html">source</a></em></small>

![Keycloak SAML role-list mapper exposing the Roles attribute.](/images/wazuh-keycloak/keycloak-saml-mapper.png)

<small><em>Exposing Keycloak roles as the SAML `Roles` attribute · Wazuh documentation · <a href="https://documentation.wazuh.com/current/user-manual/user-administration/single-sign-on/keycloak.html">source</a></em></small>

![Wazuh role mapping screen mapping the wazuh-admins backend role to administrator.](/images/wazuh-keycloak/wazuh-role-mapping.png)

<small><em>Mapping the returned `wazuh-admins` backend role in Wazuh · Wazuh documentation · <a href="https://documentation.wazuh.com/current/user-manual/user-administration/single-sign-on/keycloak.html">source</a></em></small>

## Configure Keycloak

### 1. Create a dedicated realm

Create a realm named `Wazuh`, or use an existing realm only if its lifecycle, administrators and policies are already suitable for the security platform. A dedicated realm keeps the integration’s client, groups, roles and authentication flow easier to audit.

### 2. Create the SAML client

In **Clients**, create a client with:

- **Client type:** `SAML`
- **Client ID:** `wazuh-saml`
- **Name:** `Wazuh SSO`

The client ID is used later as the Wazuh Service Provider entity ID. It is not a secret.

### 3. Configure the SAML endpoints and signatures

Use the real dashboard URL in every field. The Wazuh documentation uses these values as a baseline:

| Keycloak setting | Value |
| --- | --- |
| Valid redirect URIs | `https://<WAZUH_DASHBOARD_URL>/*` |
| IDP-Initiated SSO URL name | `wazuh-dashboard` |
| Name ID format | `username` |
| Force POST binding | On |
| Include AuthnStatement | On |
| Sign documents | On |
| Sign assertions | On |
| Signature algorithm | `RSA_SHA256` |
| SAML signature key name | `KEY_ID` |
| Canonicalization method | `EXCLUSIVE` |
| Front-channel logout | On |
| Client signature required | Off, as in the Wazuh integration guide |

In a production deployment, narrow redirect and logout URLs to the exact dashboard origin and paths supported by the installed versions. A broad wildcard is convenient for a lab but increases the impact of a client-configuration mistake.

Under **Advanced → Fine Grain SAML Endpoint Configuration**, configure:

```text
Assertion Consumer Service POST Binding URL:
https://<WAZUH_DASHBOARD_URL>/_opendistro/_security/saml/acs/idpinitiated

Logout Service Redirect Binding URL:
https://<WAZUH_DASHBOARD_URL>
```

### 4. Create the roles

Create at least two realm roles:

```text
wazuh-admins
wazuh-readonly
```

Use groups to assign the roles where possible. For example, a `Wazuh administrators` group can receive `wazuh-admins`, while a `Wazuh analysts` group receives `wazuh-readonly`. This keeps access reviews focused on group membership instead of individual exceptions.

Create test users in those groups. Set passwords through Keycloak’s credential workflow, require MFA according to the realm policy, and avoid using the built-in master realm administrator as a Wazuh test identity.

### 5. Expose the role in the SAML assertion

In the `role_list` client scope, add a **Role list** protocol mapper with:

```text
Mapper type: Role list
Name: wazuhRoleKey
Role attribute name: Roles
SAML Attribute NameFormat: Basic
Single Role Attribute: On
```

Ensure that the client receives the scope containing this mapper. The spelling and case are significant: `Roles`, `roles` and `backend_roles` are not interchangeable values.

### 6. Download the metadata

From the `wazuh-saml` client, use **Action → Download adapter config** and select the `mod-auth-mellon` format. The downloaded package contains the IdP and Service Provider metadata files.

The IdP entity ID must be copied from `idp-metadata.xml` rather than guessed from the realm URL. Metadata is the contract used by the Wazuh indexer to locate and trust the IdP endpoints and signing certificate.

## Configure the Wazuh indexer

The indexer validates the SAML assertion and maps the backend role to indexer permissions.

### 1. Back up the security configuration

On the Wazuh indexer node, create a backup before editing the security configuration:

```bash
sudo /usr/share/wazuh-indexer/bin/indexer-security-init.sh \
  --options "-backup /etc/wazuh-indexer/opensearch-security -icl -nhnv"
```

Copy `idp-metadata.xml` and `sp-metadata.xml` into:

```text
/etc/wazuh-indexer/opensearch-security/
```

Set ownership for the Wazuh indexer service account:

```bash
sudo chown wazuh-indexer:wazuh-indexer \
  /etc/wazuh-indexer/opensearch-security/idp-metadata.xml
sudo chown wazuh-indexer:wazuh-indexer \
  /etc/wazuh-indexer/opensearch-security/sp-metadata.xml
```

### 2. Generate the exchange key

The `exchange_key` is used by the SAML authentication integration. Generate a fresh value for the environment and keep it out of source control:

```bash
COMMAND_NAME="wazuh-saml-exchange-key"
echo "${COMMAND_NAME}: generate a lab-only exchange key"
openssl rand -hex 32
```

The output is 64 hexadecimal characters. Treat it as sensitive configuration even though it is not a user password.

### 3. Add the SAML authentication domain

Edit `/etc/wazuh-indexer/opensearch-security/config.yml`. Keep the internal basic-auth domain as a non-challenging fallback and add the SAML domain with the values from your metadata:

```yaml
authc:
  basic_internal_auth_domain:
    description: "Authenticate via HTTP Basic against internal users database"
    http_enabled: true
    transport_enabled: true
    order: 0
    http_authenticator:
      type: "basic"
      challenge: false
    authentication_backend:
      type: "intern"

  saml_auth_domain:
    http_enabled: true
    transport_enabled: false
    order: 1
    http_authenticator:
      type: saml
      challenge: true
      config:
        idp:
          metadata_file: "/etc/wazuh-indexer/opensearch-security/idp-metadata.xml"
          entity_id: "<IDP_ENTITY_ID_FROM_METADATA>"
        sp:
          entity_id: "wazuh-saml"
          metadata_file: "/etc/wazuh-indexer/opensearch-security/sp-metadata.xml"
        kibana_url: "https://<WAZUH_DASHBOARD_URL>"
        roles_key: "Roles"
        exchange_key: "<64_CHARACTER_EXCHANGE_KEY>"
    authentication_backend:
      type: noop
```

Do not copy the example key into production. Replace the IdP entity ID, dashboard URL and metadata paths with the values from the actual environment. Keep the exact case of the `roles_key` aligned with the SAML mapper.

### 4. Apply the indexer configuration

Use the Wazuh-provided `securityadmin.sh` command with the correct administrator certificates for the node:

```bash
sudo env JAVA_HOME=/usr/share/wazuh-indexer/jdk/ \
  bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
  -f /etc/wazuh-indexer/opensearch-security/config.yml \
  -icl \
  -key /etc/wazuh-indexer/certs/admin-key.pem \
  -cert /etc/wazuh-indexer/certs/admin.pem \
  -cacert /etc/wazuh-indexer/certs/root-ca.pem \
  -h localhost \
  -nhnv
```

The command must complete successfully and update the `config` type. If it fails, stop there and restore the backup or correct the configuration; do not continue by weakening certificate or signature validation.

### 5. Map the administrator backend role

Edit `/etc/wazuh-indexer/opensearch-security/roles_mapping.yml` and map the Keycloak backend role to the appropriate indexer role:

```yaml
all_access:
  reserved: false
  hidden: false
  backend_roles:
    - "wazuh-admins"
```

Apply the mapping with the same `securityadmin.sh` tool, changing the `-f` argument to `roles_mapping.yml`. Use a dedicated least-privilege indexer role for analysts instead of assigning `all_access` to every authenticated user.

## Configure the Wazuh dashboard

### 1. Preserve the Wazuh API identity

Verify that `/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml` keeps `run_as: true` for the dashboard’s Wazuh API connection:

```yaml
hosts:
  - default:
      url: https://localhost
      port: 55000
      username: wazuh-wui
      password: "<WAZUH_WUI_PASSWORD>"
      run_as: true
```

This setting allows the dashboard to create the Wazuh-side role mapping for the authenticated identity. It is separate from the credentials used by a human operator.

### 2. Enable SAML authentication in the dashboard

Edit `/etc/wazuh-dashboard/opensearch_dashboards.yml` and add the SAML authentication settings:

```yaml
opensearch_security.auth.multiple_auth_enabled: true
opensearch_security.auth.type: ["basicauth", "saml"]
server.xsrf.allowlist:
  - "/_opendistro/_security/saml/acs"
  - "/_opendistro/_security/saml/logout"
  - "/_opendistro/_security/saml/acs/idpinitiated"
```

Keep basic authentication enabled until SAML has been tested. The internal path is useful as a break-glass mechanism, but its credentials must be vaulted, monitored and excluded from routine use.

Restart the dashboard:

```bash
sudo systemctl restart wazuh-dashboard
```

### 3. Create the Wazuh role mapping

Log in to the dashboard with a local administrator account and open **Server management → Security → Roles mapping**. Create a mapping with:

| Field | Value |
| --- | --- |
| Roles | `administrator` |
| User field | `backend_roles` |
| Search operation | `FIND` |
| Value | `wazuh-admins` |

For the analyst path, map `wazuh-readonly` to a read-only Wazuh role. Do not rely on a successful Keycloak login as proof of authorization: test an action that must be allowed and an action that must be denied.

## Validate the complete flow

Test with a clean browser session or a private window so that previous SAML cookies do not hide configuration errors.

### Authentication test

1. Open the Wazuh dashboard URL.
2. Select the SAML or Keycloak login path.
3. Authenticate with the test user in the `Wazuh` realm.
4. Confirm that the browser returns to the Wazuh dashboard.
5. Confirm the displayed user and tenant are the expected ones.

### Authorization test

For `wazuh-admins`, verify an administrative action in a disposable environment. For `wazuh-readonly`, verify that the user can read the intended dashboards and events but cannot edit rules, manage users, change agents or alter indexer security.

### Assertion test

Inspect the Wazuh indexer and dashboard logs around the login. Confirm that:

- the assertion issuer matches the IdP metadata;
- the signature validates against the expected Keycloak certificate;
- the `Roles` attribute is present with the expected value;
- the backend role is mapped in both the indexer and the Wazuh dashboard;
- logout removes the application session as expected.

Avoid logging raw SAML assertions in a shared environment. Assertions can contain usernames, group membership and other identity attributes.

## Troubleshooting by symptom

| Symptom | Likely area to inspect |
| --- | --- |
| Redirect URI or ACS error | Dashboard URL, ACS path, client binding and Keycloak endpoint settings |
| Invalid issuer | `idp.entity_id` versus the value in `idp-metadata.xml` |
| Signature validation failure | Metadata file, Keycloak signing certificate, trust chain or clock skew |
| Login succeeds but access is denied | `Roles` attribute, backend role spelling and Wazuh role mapping |
| User is authenticated as the wrong role | Group membership, role scope and mapper assignment |
| Login loop | SAML binding, XSRF allowlist, cookies, proxy headers or dashboard URL |
| Dashboard loads but API actions fail | `run_as`, Wazuh API credentials or dashboard-side role mapping |
| Works once, then fails after time | Certificate rotation, metadata refresh or time synchronization |

The most common integration mistake is solving authentication while forgetting authorization. If the assertion is valid but the user has no effective backend role, the federation is working; the mapping is not.

## Hardening the integration

### Use least privilege as the default

Create the read-only path first. Add administrative access only to a small group, review it periodically and avoid assigning `all_access` to a broad Keycloak role.

### Narrow the trust boundaries

Use HTTPS end to end, restrict the dashboard’s network exposure, and keep the Keycloak realm and Wazuh dashboard clocks synchronized. If a reverse proxy terminates TLS, validate the forwarded host and scheme so generated SAML URLs remain correct.

### Treat metadata and certificates as lifecycle objects

Track the Keycloak signing certificate expiration, define a rotation procedure and test rollover in a lab. Do not replace signature validation with a permissive trust setting to make a failed login work.

### Keep a controlled break-glass path

Retain one local administrator account outside normal SSO use. Store its credentials in a vault, restrict who can retrieve them, alert on use and test the recovery procedure. Removing the fallback before testing SAML can turn a configuration error into an administrative lockout.

### Make role mapping auditable

Version the sanitized configuration, record the owner of each mapping and review the relationship:

```text
Keycloak group → Keycloak role → SAML Roles attribute
→ Wazuh backend role → indexer / dashboard permissions
```

This chain should be understandable to both the IAM and SOC teams. An access review that checks only Keycloak group membership is incomplete if Wazuh maps the resulting role differently.

## What this integration demonstrates

Wazuh provides security visibility, but its own dashboard is a security-sensitive application. Connecting it to Keycloak makes the identity lifecycle easier to centralize, strengthens MFA and reduces local-user sprawl. It also creates a new dependency: the SOC must be able to authenticate when Keycloak, its network path, metadata or signing certificate is unavailable.

The right design is therefore not “SSO everywhere at any cost”. It is explicit federation, narrow role mappings, a protected recovery path and tests that cover both authentication and authorization. In a security platform, a successful login is only the beginning of the control being tested.

## Sources and image credits

1. [Wazuh — Keycloak Single Sign-On](https://documentation.wazuh.com/current/user-manual/user-administration/single-sign-on/keycloak.html), official integration guide.
2. [Wazuh — Single Sign-On](https://documentation.wazuh.com/current/user-manual/user-administration/single-sign-on/index.html), supported identity-provider and SAML concepts.
3. [Wazuh — Architecture](https://documentation.wazuh.com/current/getting-started/architecture.html), component responsibilities.
4. [Wazuh — Wazuh dashboard](https://documentation.wazuh.com/current/getting-started/components/wazuh-dashboard.html), dashboard capabilities.
5. [Keycloak Server Administration Guide](https://www.keycloak.org/docs/latest/server_admin/), realm and client administration.
6. Screenshots: [Wazuh Keycloak integration documentation](https://documentation.wazuh.com/current/user-manual/user-administration/single-sign-on/keycloak.html), reproduced with attribution for instructional context.
