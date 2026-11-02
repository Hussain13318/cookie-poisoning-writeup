# Hack the Cookie: Cookie Poisoning Attack Writeup

**Lab:** https://hackerdna.com/labs/hack-the-cookie
**Difficulty:** Very Easy
**Category:** Web & API Security

## Overview

This lab demonstrates a classic **cookie poisoning / insecure session management** vulnerability, where a web application stores sensitive authorization data (like a user's role) client-side in a cookie, without any cryptographic protection such as a signature (e.g., HMAC) or server-side session validation. Because the cookie is just base64-encoded  not encrypted or signed  an attacker can freely read and modify it.

## Methodology

### 1. Recon
Opened the target application and logged in as the default `guest` user. Opened Chrome DevTools → **Application** tab → **Cookies** to inspect what the app was storing client-side.

A cookie named `user_session` stood out  its value was a long base64-looking string rather than an opaque session token (like a UUID or random hash), which is usually a red flag for this class of vulnerability.

### 2. Decoding the Cookie
Base64-decoding the `user_session` value revealed a plaintext JSON object containing user attributes, including a `role` field set to `"guest"`. No HMAC signature, nonce, or server-side reference was present  meaning the application trusted the cookie's contents at face value.

### 3. Tampering
I modified the `role` field in the decoded JSON from `guest` to `admin`, keeping the rest of the structure identical, then re-encoded the modified JSON back to base64.

### 4. Replaying the Forged Cookie
Using DevTools' Application → Cookies editor, I replaced the original `user_session` value with my newly crafted, admin-role cookie, then refreshed the page.

### 5. Privilege Escalation Confirmed
The application accepted the forged cookie without any server-side validation, granting access to an admin-only area that was previously inaccessible as a guest user. The flag was located within this restricted admin view.

## Root Cause

The vulnerability stems from the application:
- Storing authorization-sensitive data (`role`) directly in a client-readable/writable cookie
- Failing to sign or encrypt the session payload
- Performing no server-side verification of the role against a trusted data store (e.g., a database lookup by session ID)

This is a textbook violation of the principle that **the client should never be trusted to self-report its own privilege level.**

## Tools Used
- Chrome DevTools (Application tab, Cookie editor)
- Base64 decode/encode (browser console `atob()`/`btoa()`, or any standard base64 utility)

## Remediation / Lessons Learned
- Use opaque, random session identifiers that reference server-side session state  never embed roles or permissions directly in client-controlled cookies.
- If session data must be client-side (e.g., JWTs), sign it with a server-held secret (HMAC/RSA) and verify the signature on every request.
- Set cookies with `HttpOnly`, `Secure`, and `SameSite` flags to reduce exposure, though this alone wouldn't prevent this specific attack since the issue is tamperability, not just visibility.
- Always re-validate authorization server-side; never trust client-supplied role/permission claims.

## Conclusion
This lab is a great beginner-friendly illustration of why **trust boundaries matter**: anything stored or transmitted via the client (cookies, hidden form fields, localStorage) must be treated as untrusted input and validated server-side before being used for authorization decisions.

---
*Lab reference: https://hackerdna.com/labs/hack-the-cookie*
