# LAB 15 – Breaking SSL Pinning on Android  
## Dynamic Analysis | TLS/HTTPS Interception | Frida Mayhem

---

> Omayma El Yamani  
---

##  Mission Objectives

- Deploy Frida (host + device) like a ghost
- Force Android traffic through a rogue TLS proxy
- Obliterate SSL Pinning using JavaScript runtime injection
- Capture decrypted HTTPS traffic in plain sight
- Compare Java-level vs native-level pinning mechanisms

---


##  Environment Setup Check

| Component | Status | Version |
|-----------|--------|---------|
| Frida Client | ✅ | 17.9.5 |
| frida-server (Android) | ✅ | 17.9.5 |
| ADB Connection | ✅ | emulator-5554 |
| Burp Suite | ✅ | Listener :8080 |

---

## Step 1 

<img width="666" height="479" alt="image" src="https://github.com/user-attachments/assets/16ca9a7d-c725-4ee2-88f8-c89659414222" />

The learning platform interface showing all 12 steps of SSL pinning bypass methodology.

---

##  Step 2 – Frida Verification

<img width="537" height="85" alt="image" src="https://github.com/user-attachments/assets/945075e1-0033-4907-b89a-0440757837b7" />

```bash
frida --version              → 17.9.5
python -c "import frida; print(frida.__version__)" → 17.9.5
```
Perfect alignment between client and server versions — mandatory for stable instrumentation.

##  Step 3 – Reconnaissance on Target Device
<img width="502" height="214" alt="image" src="https://github.com/user-attachments/assets/4220d23f-a876-479b-b4b3-2650e62c58e3" />

```bash
frida-ps -U
```
Visible processes discovered:

- 14318 → Calendar

- 14153 → Chrome

- 12507 → ConverterTabsJava

- 11290 → FragmentsLab

- 13718 → JNIDemo

This confirms Frida's ability to attach to any running Java process.

## Step 4 – Proxy Listener Configuration

<img width="518" height="175" src="https://github.com/user-attachments/assets/4d599f38-2ddf-4c0e-bb89-301d1a8ebedd" />

Burp Suite is configured with a listener on *:8080.
This allows interception of all HTTP/HTTPS traffic from the Android emulator.

Pro tip: Enable "Invisible" proxy mode to avoid detection by certificate pinning.

## Step 5 – Android Network Redirection
<img width="184" height="375" src="https://github.com/user-attachments/assets/55b95ee0-e7c5-4781-993f-bce3cb90e0d4" />

The emulator's Wi-Fi is configured to route all traffic through the host's Burp proxy.
Security set to "None" — no encryption, just raw forwarding.

Without this step, the phone ignores the proxy completely.

## Step 6 – First Strike: Uncrackable 3
<img width="680" height="345" alt="image" src="https://github.com/user-attachments/assets/55b0d80f-54e1-4604-a98b-167befb6915b" />

```bash
frida -U -f owasp.mstg.uncrackable3 -l sslpin_bypass_universal.js
```
Observed output:

```text
[+] SSL bypass: SSLContext.init patched
[+] SSL bypass: X509TrustManager patches attempted
[+] SSL bypass: com.android.org.conscrypt.TrustManagerImpl patched
[+] Universal SSL pinning bypass installed
```
✅ Verdict: Full Java‑level pinning destroyed.
The app now accepts any certificate — including Burp's CA.

## Step 7 – Second Strike: Uncrackable 2
<img width="693" height="233" alt="image" src="https://github.com/user-attachments/assets/59b4d073-0c1e-41ee-ba94-7cbb28454bd6" />

```bash
frida -U -f owasp.mstg.uncrackable2 -l sslpin_bypass_universal.js
```
Observed output:

```text
[*] Scanning loaded SSL / network classes...
[+] javax.net.ssl.SSLSessionBindingEvent
```
⚠️ Verdict:
Uncrackable 2 does not expose classic TrustManager hooks.
Possible explanations:

- Custom native pinning (BoringSSL / OpenSSL)

- Cronet or embedded Chromium network stack

- Obfuscated pinning logic

> Conclusion: Java hooks are insufficient here. Native tracing required.

## Advanced Native Pinning Bypass 
When Java hooks fail, go native:

```javascript
// sslpin_bypass_native.js
function hookNativeSSL() {
    var addr = Module.findExportByName("libssl.so", "SSL_get_verify_result");
    if (addr) {
        Interceptor.attach(addr, {
            onLeave(retval) {
                console.log("[+] Overriding SSL_get_verify_result → X509_V_OK");
                retval.replace(ptr(0));
            }
        });
    }
}
```
Combine both scripts for maximum destruction:

```bash
frida -U -f owasp.mstg.uncrackable2 -l sslpin_bypass_universal.js -l sslpin_bypass_native.js
```

## Validation Matrix

| Test Case | Expected Result | Actual Status |
|-----------|-----------------|---------------|
| Frida spawn without crash | ✅ | ✅ |
| Bypass logs visible | ✅ | ✅ (Uncrackable 3) |
| HTTPS traffic in Burp | ✅ | ✅ (Uncrackable 3) |
| Uncrackable 2 full bypass | ⚠️ | Partial (native needed) |

## Final Thoughts

| Target | Pinning Type | Status |
|--------|--------------|--------|
| Uncrackable 3 | Java (TrustManager) | ✅ Fully broken |
| Uncrackable 2 | Native (likely OpenSSL) | ⚠️ Requires native hooks |

"Every lock has a key. Sometimes the key is JavaScript. Sometimes it's C. Frida speaks both."
— Omayma El Yamani, LAB 15
