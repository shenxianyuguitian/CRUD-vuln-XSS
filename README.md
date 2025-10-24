# CRUD-vuln-XSS
Proof-of-Concept and Advisory for Simple CRUD XSS

##Vulnerability Advisory & Exploit
#Affected Version

SIMPLE_CRUD_IN_CODEIGNITER_USING_VUE.JS (civuejs) — demo CodeIgniter + Vue.js CRUD project (user management UI using v-html for server messages)

#Vulnerability Type

DOM-based Cross-Site Scripting (DOM XSS) — client blindly assigns server JSON (response.data.msg) into Vue v-html bindings (e.g., v-html="formValidate.firstname"), allowing an attacker who can influence an AJAX response to inject and execute script in the victim’s browser.

#Advisory (Recommendations)

Replace v-html usage for server-supplied validation/error fields with safe interpolation ({{ ... }}) or v-text.

If rendering HTML is required, sanitize server-supplied HTML using a strict whitelist (server-side HTMLPurifier) and also sanitize on the client (DOMPurify) before assigning to v-html.

Ensure authentication and response integrity: require authentication for pages that submit/receive user data, enforce CSRF protection, and implement server-side checks so untrusted actors cannot tamper with JSON endpoints.

Use HTTPS and HSTS to protect against network tampering; consider additional transport integrity (pinning) or response signing in high-risk deployments.

Add Content Security Policy (CSP) restricting inline script execution (e.g., script-src 'self') to reduce impact.

#Summary

A client-side vulnerability exists because the frontend directly uses v-html to render server-provided JSON fields. An attacker who can alter the AJAX response (for example via a man-in-the-middle in an untrusted network, or through a compromised reverse proxy) can inject HTML/JS into the msg fields; the frontend will insert that HTML into the DOM and execute scripts. This is a DOM XSS vector demonstrating how untrusted JSON rendered with v-html becomes an execution sink.

#Root Cause

The Vue component assigns server response.data.msg into a reactive object and templates render fields with v-html. There is no client- or server-side sanitization of msg fields prior to insertion into the DOM, and the application may not ensure transport integrity for the AJAX response.

#Impact

 Script execution in victims’ browsers when they view the modal/page that receives the tampered response.

 Potential for session theft, forced actions (CSRF), keylogging, or further client-side pivoting depending on user privileges and available functionality.

 If administrators view affected pages, impact is higher (credentials, account takeover).

Severity: High (depends on ability to tamper responses and which users view the page).

#Proof-of-Concept (non-destructive, controlled-environment PoC)

Goal: Demonstrate that modifying the JSON AJAX response to include HTML/JS in msg.firstname results in script execution via v-html.


1.Open the application’s Users page in a browser in a test environment (where you can safely intercept traffic).

2.Open Developer Tools → Network, or configure an intercepting proxy (e.g., Burp/ZAP) and set the browser to use it. Ensure you have permission to test.

3.Trigger the client action that sends the AJAX POST to user/addUser (open the modal and submit the form to produce a validation response). The client will issue a JSON response like:
    
    {
      "error": true,
      "msg": {
        "firstname": "...",
        "lastname": "...",
        ...
      }
    }


4.Intercept that response in the proxy and modify the JSON msg.firstname value to include an inert, low-impact proof string representing executable HTML — for safe confirmation prefer a non-alert variant that logs to console, e.g.:

    {
      "error": true,
      "msg": {
        "firstname": "<img src=x onerror=console.log('XSS-TEST')>",
        "lastname": "",
        ...
      }
    }


5.Forward the modified response to the browser. The client code will perform v.formValidate = response.data.msg and the modal template containing v-html="formValidate.firstname" will render the injected HTML.

6.Verify in the browser console that XSS-TEST is logged — this demonstrates script execution arising from the tampered JSON response.

Why this proves the issue: The injected img with onerror is executed only when the application inserts the modified HTML into the DOM without sanitization. Using console.log avoids disruptive pop-ups while proving execution.

#Mitigations & Quick Fix Snippets

 Client change (preferred): stop using v-html for server validation messages:

    <!-- unsafe -->
    <div class="has-text-danger" v-html="formValidate.firstname"></div>
    
    <!-- safe -->
    <div class="has-text-danger" v-text="formValidate.firstname"></div>
    <!-- or -->
    <div class="has-text-danger">{{ formValidate.firstname }}</div>


 If HTML must be shown: sanitize before assigning:

  // client-side example with DOMPurify
  import DOMPurify from 'dompurify'
  response.data.msg.firstname = DOMPurify.sanitize(response.data.msg.firstname, {ALLOWED_TAGS: [], ALLOWED_ATTR: []})
  this.formValidate = response.data.msg


Server: ensure all JSON responses contain only safe strings (escape or sanitize on server), and enable CSRF and transport security (HTTPS). Consider signing responses if tamper-resistance is required.
