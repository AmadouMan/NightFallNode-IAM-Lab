# NightFallNode IAM Lab — Keycloak IdP → Wazuh SIEM

A self-hosted, four-part **Identity & Access Management + Detection Engineering** lab, built end to end in a home SOC. It walks the full identity lifecycle — **build the IdP → enforce authorization → detect the violation → federate identity** — and closes the loop by detecting an offboarded account from the SIEM.

> Built and streamed live by **Amadou MANE** — Detection and Response Engineer.
> 📺 Live labs on [Twitch](https://www.twitch.tv/cyberops401unauthorized) · ▶️ Full series on [YouTube](https://www.youtube.com/@cyberops401unauthorized)

---

## Lab topology

| VM | Role | Stack |
|----|------|-------|
| **SecMon** | Monitoring + IdP | Keycloak 26.8.1, Wazuh 4.14.5 (manager, indexer, dashboard) |
| **NexusShield-Corp** | Corporate endpoint / directory | Ubuntu, OpenLDAP (slapd) |
| **DayGone** | Attacker | Kali Linux |

```
NexusShield-Corp (.133)          SecMon (.131)                 DayGone (.135)
  OpenLDAP directory   <--LDAP-->  Keycloak (IdP, federation)    attacker
  corp employees                   Wazuh SIEM  <---- events ---- 
```

---

## The labs

| # | Lab | What it covers | Key result |
|---|-----|----------------|------------|
| **1** | **Build your own Identity Provider** | Native Keycloak install (systemd), realms, OIDC client, Implicit vs **Authorization Code + PKCE**, full JWT claim analysis | A working IdP; tokens decoded claim by claim |
| **2** | **Authentication ≠ Authorization** | App role granted to groups, a custom `gated-browser` conditional flow bound to one client | Contractor denied with a **valid password** — even on an existing SSO session |
| **3** | **From Identity Event to SIEM Alert** | Keycloak JSON event log → Wazuh `localfile` → custom rules (base → authz-failure → correlation) | Level-10 correlation alert on repeated denials, **MITRE T1078 / T1087** |
| **4** | **User Federation + Offboarding Detection** | Federate Keycloak to OpenLDAP (READ_ONLY), map LDAP groups to roles, offboard a user, detect the return | Federated user authorized with **zero per-user config**; offboarded account's login attempt detected (**T1110**) |

---

## Detection ruleset (Wazuh `local_rules.xml`)

Custom rules under the `nightfallnode_soc` group, each mapped to MITRE ATT&CK and organised by incident ID (INC-xxx). Identity-focused rules:

| Rule | Incident | Fires on | Meaning | MITRE |
|------|----------|----------|---------|-------|
| `100018` | INC-016 | `LOGIN_ERROR` + `access_denied` | Authorization denial / insider probing | T1078 |
| `100019` | INC-016 | 3× `100018` in 300s | Repeated denials = probing | T1078, T1087 |
| `100020` | INC-017 | `LOGIN_ERROR` + `invalid_user_credentials` | Offboarded user returns / failed auth | T1110 |
| `100021` | INC-017 | 5× `100020` in 60s | Brute force / credential stuffing | T1110 |

The broader lab ruleset also covers SSH/FTP brute force, web app scanning, SQLi, XSS, directory traversal, reverse shells, privilege escalation, log tampering, data exfiltration, malware, persistence, and C2 — 21 rules total, all MITRE-mapped.

---

## Security lessons captured

- **Authentication ≠ authorization.** A valid login is not permission. Denials happen at different layers (wrong group → authz; deleted user → authn).
- **Enforce at every path.** An access check that lives only in the login form is bypassed by an existing SSO session — it must run after every authentication path.
- **One source of truth.** Federating to the directory means onboarding/offboarding happen once and propagate everywhere.
- **Offboarding has a gap.** Deleting an account blocks new logins, but a token already issued survives until expiry — proper deprovisioning must also revoke active sessions.
- **Know your log source.** The password-path denial logs the username; the SSO-path denial does not — correlate by `code_id` and source IP.

---

## Tech stack

`Keycloak` · `OpenLDAP` · `Wazuh SIEM` · `OIDC / OAuth2` · `SAML` · `PKCE` · `JWT` · `MITRE ATT&CK` · `Linux (systemd)` · `JSON log pipelines` · `Detection Engineering`

---

## Watch the labs

- ▶️ **Lab 4 — User Federation:** https://youtu.be/cWRETP3aNO0
- ▶️ **Lab 5 — Offboarding & SOC Detection:** https://youtu.be/ODI7L3KfO2Q
- 📺 **Full series & live labs:** https://www.youtube.com/@cyberops401unauthorized

---

## Suggested repo structure

```
NightFallNode-IAM-Lab/
├── README.md
├── docs/                 # lab write-ups (PDF)
│   ├── Lab1_Keycloak_OIDC.pdf
│   ├── Lab2_Authn_vs_Authz.pdf
│   ├── Lab3_Keycloak_to_Wazuh.pdf
│   └── Lab4_Federation_and_Detection.pdf
├── keycloak/
│   ├── keycloak.service          # systemd unit
│   └── gated-browser-flow.md     # authentication flow design
├── ldap/
│   └── seed.ldif                 # directory users & groups
└── wazuh/
    ├── local_rules.xml           # custom detection rules
    └── ossec.conf.snippet        # localfile block for Keycloak log
```

---

*Detection and Response Engineering — identity, from IdP to SIEM.*
