
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

A black-box penetration test was conducted against the Recruit web application to assess its security posture.

The assessment identified multiple vulnerabilities that could be chained together to obtain full administrative access. Initial reconnaissance revealed publicly accessible deployment artifacts, followed by a Local File Inclusion vulnerability that disclosed application configuration files containing HR credentials. A SQL Injection vulnerability then enabled extraction of administrator credentials directly from the backend database.

The demonstrated attack path resulted in complete compromise of the administrative account.

| Metric          | Value                   |
| --------------- | ----------------------- |
| Assessment Type | Black Box               |
| Target          | Recruit Web Application |
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
![[Pasted image 20260627155840.png]]

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
![[Pasted image 20260627155930.png|605]]

---

## Mail Enumeration

A publicly accessible mail archive disclosed deployment information stating:

- HR credentials were temporarily stored in `config.php`
    
- Administrator credentials were stored in the backend database
    

### Screenshot
![[Pasted image 20260627160241.png]]

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
![[Pasted image 20260627160234.png]]
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
![[Screenshot 2026-06-27 145847 2.png]]

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
![[Pasted image 20260627172416.png]]

---

## Column Enumeration

```sql
%' UNION SELECT 1,2,3,4-- -
```

Result:

- 4 columns identified.
    

### Screenshot
![[Pasted image 20260627172500.png]]

---

## Table Enumeration

```sql
%' UNION SELECT 1,table_name,3,4
FROM information_schema.tables
WHERE table_schema=database()-- -
```

### Screenshot
![](file:///C:/Users/Swaraj/Downloads/Screenshot%202026-06-27%20172519.png)

---

## Credential Extraction

```sql
%' UNION SELECT 1,username,password,4
FROM users-- -
```

### Screenshot
![](file:///C:/Users/Swaraj/Downloads/Screenshot%202026-06-27%20172618.jpg)

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

### Screenshot
![[Pasted image 20260627172903.png]]

---

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

The Recruit web application contains multiple vulnerabilities that can be chained together to achieve complete administrative compromise.

The assessment demonstrated a realistic attack path beginning with information disclosure, progressing through Local File Inclusion, and culminating in SQL Injection that enabled extraction of administrator credentials.

Immediate remediation is recommended, with priority given to the SQL Injection and Local File Inclusion vulnerabilities due to their direct impact on authentication and sensitive data exposure.