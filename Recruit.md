# Recruit - Writeup
> [Recruit Web Challenge](https://tryhackme.com/room/recruitwebchallenge)
---
# Table of Contents

- Executive Summary
    
- Scope
    
- Methodology
    
- Reconnaissance
    
- Vulnerability Assessment
    
    - F-01 Information Disclosure
        
    - F-02 Local File Inclusion
        
    - F-03 Hardcoded Credentials
        
    - F-04 SQL Injection
        
    - F-05 Missing HttpOnly Cookie Attribute
        
- Attack Chain
    
- Risk Summary
    
- Recommendations
    
- Conclusion
    

---

# Executive Summary

The assessment identified multiple vulnerabilities that could be chained together to obtain full administrative access. Initial reconnaissance revealed publicly accessible deployment artifacts, followed by a Local File Inclusion vulnerability that disclosed application configuration files containing HR credentials. A SQL Injection vulnerability then enabled extraction of administrator credentials directly from the backend database.

The demonstrated attack path resulted in complete compromise of the administrative account.

| Metric          | Value                   |
| --------------- | ----------------------- |
| Assessment Type | Black Box               |
| Target          | Recruit THMLAB          |
| Overall Risk    | Critical                |

---

# Scope

| Item             | Value                            |
| ---------------- | -------------------------------- |
| Target IP        | `10.48.158.217`                  |
| Operating System | Ubuntu Linux                     |
| Web Server       | Apache 2.4.41                    |
| Test Type        | Web Application Penetration Test |

---

# Methodology

The assessment followed the OWASP Web Security Testing Guide (WSTG) methodology.

Testing included:

- Information Gathering
    
- Service Enumeration
    
- Content Discovery
    
- Authentication Testing
    
- Input Validation Testing
    
- SQL Injection Testing
    
- Privilege Escalation
    
- Authentication Verification
    

---

# Reconnaissance

## Service Enumeration

An Nmap scan identified the following services.

|Port|Service|Version|
|--:|---|---|
|22|SSH|OpenSSH 8.2p1|
|53|DNS|ISC BIND 9.16.1|
|80|HTTP|Apache 2.4.41|

### Screenshot
<img width="1351" height="541" alt="image" src="https://github.com/user-attachments/assets/a182fc00-c769-48a3-9f18-e225c3240c0a" />


---

## Sitemap Enumeration

The application exposed a public sitemap containing several interesting endpoints.

```text
/api.php
/file.php
/mail/
/dashboard.php
```

### Screenshot
<img width="842" height="893" alt="image" src="https://github.com/user-attachments/assets/60a21fba-a773-4cc6-bd07-f613b95de55c" />


---

## Mail Enumeration

A publicly accessible mail archive disclosed deployment information stating:

- HR credentials were temporarily stored in `config.php`
    
- Administrator credentials were stored in the backend database
    

### Screenshot
<img width="1318" height="846" alt="image" src="https://github.com/user-attachments/assets/45f8302f-6a48-4fc4-a1ac-83e754e87eff" />


---

# Vulnerability Assessment

---

# F-01 Information Disclosure

**Severity:** Medium

## Description

Internal deployment emails were publicly accessible through the web application.

The disclosed information significantly reduced the effort required during reconnaissance by revealing credential storage locations and application architecture.

## Evidence

**Affected Resource**

```text
/mail/
```

### Screenshot
<img width="1318" height="846" alt="image" src="https://github.com/user-attachments/assets/41eb2b3a-db07-4654-aeff-f2f2ed7efb8d" />

## Impact

- Information disclosure
    
- Facilitates further attacks
    
- Reduces attack complexity
    

## Recommendation

- Remove public access to internal mail archives.
    
- Store operational communications securely.
    
- Avoid exposing deployment details.
    

---

# F-02 Local File Inclusion (LFI)

**Severity:** High

## Description

The file retrieval endpoint failed to validate user-controlled input, allowing directory traversal and arbitrary file disclosure.

## Proof of Concept

```text
/file.php?cv=file:///var/www/html/config.php
```

## Evidence

The application disclosed the PHP source code of the configuration file.

### Screenshot
<img width="877" height="687" alt="image" src="https://github.com/user-attachments/assets/0013f691-b57a-4c42-97b4-5106ec651f2a" />


The configuration contained:

- HR credentials
    
- Application configuration
    
- API configuration
    

## Impact

- Credential disclosure
    
- Source code disclosure
    
- Sensitive information exposure
    

## Recommendation

- Restrict file access to approved directories.
    
- Validate file paths.
    
- Implement allow-listed filenames.
    

---

# F-03 Hardcoded Credentials

**Severity:** Medium

## Description

Application credentials were embedded directly within the production configuration file.

## Impact

Any successful file disclosure immediately exposes privileged credentials.

## Recommendation

- Remove hardcoded credentials.
    
- Store secrets using environment variables or a secrets manager.
    
- Rotate exposed passwords.
    

---

# F-04 SQL Injection

**Severity:** Critical

## Description

The authentication functionality failed to sanitize user input, resulting in a UNION-based SQL Injection vulnerability.

The vulnerability permitted arbitrary interaction with the backend database.

---

## Validation

Authentication bypass confirmed.

```sql
%' OR 1=1-- -
```

### Screenshot
<img width="1737" height="605" alt="image" src="https://github.com/user-attachments/assets/67413dde-53fb-42ee-a603-2190d129eeeb" />


---

## Column Enumeration

```sql
%' UNION SELECT 1,2,3,4-- -
```

Result:

- 4 columns identified.
    

### Screenshot
<img width="1752" height="667" alt="image" src="https://github.com/user-attachments/assets/512212cb-5a9e-45c7-beff-570645afcac7" />


---

## Table Enumeration

```sql
%' UNION SELECT 1,table_name,3,4
FROM information_schema.tables
WHERE table_schema=database()-- -
```

### Screenshot
<img width="1657" height="657" alt="Screenshot 2026-06-27 172618" src="https://github.com/user-attachments/assets/1a9394d6-2a1c-4f82-a6e7-8df5f810551c" />

---

## Credential Extraction

```sql
%' UNION SELECT 1,username,password,4
FROM users-- -
```

### Screenshot
<img width="1787" height="708" alt="Screenshot 2026-06-27 172519" src="https://github.com/user-attachments/assets/501cd86e-c37a-412f-bd40-712d25740e9a" />

---

## Impact

Successful exploitation allows an attacker to:

- Bypass authentication
    
- Enumerate database contents
    
- Extract user credentials
    
- Obtain administrator credentials
    
- Fully compromise the application
    

## Recommendation

- Use prepared statements.
    
- Implement parameterized queries.
    
- Validate all user input.
    
- Apply least-privilege database permissions.
    

---

# F-05 Missing HttpOnly Cookie Attribute

**Severity:** Low

## Description

The application issued session cookies without the `HttpOnly` attribute.

## Impact

If a Cross-Site Scripting vulnerability were introduced, authenticated session cookies could potentially be stolen.

## Recommendation

Configure session cookies with:

```text
HttpOnly
Secure
SameSite=Lax
```

---

# Attack Chain

```text
Internet
    │
    ▼
Reconnaissance
    │
    ▼
Sitemap Enumeration
    │
    ▼
Mail Disclosure
    │
    ▼
Local File Inclusion
    │
    ▼
HR Credential Disclosure
    │
    ▼
Application Access
    │
    ▼
SQL Injection
    │
    ▼
Database Enumeration
    │
    ▼
Administrator Credential Extraction
    │
    ▼
Administrative Login
```

# Risk Summary

|ID|Finding|Severity|
|---|---|---|
|F-01|Information Disclosure|Medium|
|F-02|Local File Inclusion|High|
|F-03|Hardcoded Credentials|Medium|
|F-04|SQL Injection|Critical|
|F-05|Missing HttpOnly Cookie|Low|

---

# Recommendations

Priority remediation should focus on:

1. Eliminate SQL Injection using parameterized queries.
    
2. Patch the Local File Inclusion vulnerability.
    
3. Remove hardcoded credentials from application files.
    
4. Restrict access to internal deployment artifacts.
    
5. Configure secure session cookie attributes.
    

---

# Conclusion

The assessment demonstrated a realistic attack path beginning with information disclosure, progressing through Local File Inclusion, and culminating in SQL Injection that enabled extraction of administrator credentials.

Immediate remediation is recommended, with priority given to the SQL Injection and Local File Inclusion vulnerabilities due to their direct impact on authentication and sensitive data exposure.
