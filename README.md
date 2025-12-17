# oracle-apex-ldap-authentication

## 🔹 Overview

This document describes the steps required to enable **LDAP authentication with Oracle APEX**
using **Microsoft Active Directory**.

The solution allows Oracle Database (and APEX) to authenticate users directly against an
LDAP / Active Directory server by:

- Granting the database network access to the LDAP server
- Creating and assigning a Network ACL
- Implementing a PL/SQL function that validates credentials using `DBMS_LDAP`

This approach is useful when:

- Oracle APEX is hosted on-premises
- Active Directory is available
- Centralized authentication is required
- Windows / domain login is needed (without third-party tools)

---

## 🔹 Architecture Summary

Authentication flow:

1. Oracle Database connects to the Active Directory server over LDAP (port `389`)
2. Network ACL allows outbound LDAP communication
3. A PL/SQL function performs an LDAP bind using username/password
4. If bind succeeds → authentication is successful
5. Oracle APEX uses this function inside a custom authentication scheme

---

## 🔹 Prerequisites

Before starting, make sure you have:

- Oracle Database with `DBMS_LDAP`
- Oracle APEX installed
- Network connectivity from DB server to AD server
- LDAP / Active Directory server IP or hostname
- Valid Active Directory domain
- Database user with privileges to manage Network ACLs

---

## 🔹 Step 1: Network ACL Configuration

Oracle Database blocks outbound network access by default.  
To allow LDAP communication, a **Network ACL** must be created and assigned.

This step includes:
- Creating a new ACL
- Granting `connect` and `resolve` privileges
- Assigning the ACL to the AD server and LDAP port (`389`)

### (1) Create new ACL

```sql
BEGIN
  DBMS_NETWORK_ACL_ADMIN.CREATE_ACL(
    acl         => 'ldap_acl.xml',
    description => 'Allow LDAP Access for LDAP Authentication',
    principal   => 'HR', -- change to DB user executing LDAP call
    is_grant    => TRUE,
    privilege   => 'connect'
  );
END;
/

BEGIN
  DBMS_NETWORK_ACL_ADMIN.ADD_PRIVILEGE(
    acl       => 'ldap_acl.xml',
    principal => 'HR',
    is_grant  => TRUE,
    privilege => 'resolve'
  );
END;
/

BEGIN
  DBMS_NETWORK_ACL_ADMIN.ASSIGN_ACL(
    acl        => 'ldap_acl.xml',
    host       => '10.10.10.10', -- Active Directory server IP
    lower_port => 389,
    upper_port => 389
  );
END;
/

## ⚠️ Important Notes

principal must be the database user executing the LDAP call

The IP address must match the actual Active Directory server

Port 389 is used for standard LDAP (not LDAPS)

🔹 Step 2: LDAP Authentication Function

A PL/SQL function that authenticates users against Active Directory.

Function behavior

Initializes an LDAP session

Uses UPN format (username@domain)

Attempts a simple LDAP bind

Returns TRUE if authentication succeeds

Returns FALSE on failure

Always unbinds the LDAP session

PL/SQL Function

CREATE OR REPLACE FUNCTION FN_LDAP_SIMPLE_LOGIN (
    p_username IN VARCHAR2,
    p_password IN VARCHAR2
) RETURN BOOLEAN
AS
    l_session  DBMS_LDAP.session;
    ret        PLS_INTEGER;
    ldap_host  VARCHAR2(100) := '10.10.10.10';
    ldap_port  NUMBER := 389;
    login_id   VARCHAR2(200);
BEGIN
    DBMS_LDAP.USE_EXCEPTION := TRUE;

    -- Login format: user@domain
    login_id := LOWER(p_username) || '@yourdomain.local';

    -- Start LDAP session
    l_session := DBMS_LDAP.init(ldap_host, ldap_port);

    DBMS_LDAP.USE_EXCEPTION := FALSE;

    -- Bind using UPN
    ret := DBMS_LDAP.simple_bind_s(l_session, login_id, p_password);

    IF ret = DBMS_LDAP.SUCCESS THEN
        DBMS_LDAP.unbind_s(l_session);
        RETURN TRUE;
    ELSE
        DBMS_LDAP.unbind_s(l_session);
        RETURN FALSE;
    END IF;

EXCEPTION
    WHEN OTHERS THEN
        BEGIN
            DBMS_LDAP.unbind_s(l_session);
        EXCEPTION
            WHEN OTHERS THEN NULL;
        END;
        RETURN FALSE;
END;
/

🔹 Step 3: Testing LDAP Authentication

You can test the LDAP login manually using an anonymous PL/SQL block.

This helps verify:

ACL configuration

Network connectivity

LDAP bind correctness

BEGIN
    IF FN_LDAP_SIMPLE_LOGIN(
         'windows_login_user_name',
         'password_of_windows_user'
       ) THEN
        DBMS_OUTPUT.put_line('LOGIN OK');
    ELSE
        DBMS_OUTPUT.put_line('LOGIN FAILED');
    END IF;
END;
/

🔹 Step 4: Using with Oracle APEX

After successful testing, integrate the function into Oracle APEX:

Create a Custom Authentication Scheme

Call FN_LDAP_SIMPLE_LOGIN during login

Map successful authentication to an APEX user

This allows users to log in to APEX using their Windows / Active Directory credentials.

🔹 Security Considerations

Passwords should never be logged or stored

Consider using LDAPS (port 636) in production

Grant minimum required ACL privileges

Restrict execution of the LDAP function

Use SSL certificates where possible

🔹 Notes & Limitations

Authentication is performed in real time against Active Directory

No local user synchronization is required

Network latency can affect login time

Requires database-level ACL configuration

🔹 Author Notes

This implementation provides a lightweight, database-level LDAP authentication solution
for Oracle APEX environments that require direct Active Directory integration without
additional middleware.
