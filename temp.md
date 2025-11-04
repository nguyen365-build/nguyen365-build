The error **"Could not create SSL/TLS secure channel."** in your code is a classic symptom of a failed TLS handshake. For Visa APIs (and other mTLS APIs), this is almost always due to one or more of the following:

---

## 1. **TLS Version Not Set Early Enough**

**ServicePointManager.SecurityProtocol** must be set **before** any HTTP/HTTPS/HttpClient/ServicePoint is created.  
**Move this line to the very top of your `Main` method, before any certificate or HttpClient code:**

```csharp
static void Main(string[] args)
{
    // Set TLS 1.2 before anything else!
    ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;
    // ...rest of your code...
```

---

## 2. **Certificate Issues**

- **Wrong certificate** (not for this Visa project, or not a client certificate)
- **No private key** in the PFX (must say "You have a private key..." in certmgr.msc)
- **Expired certificate** (check NotBefore/NotAfter)
- **Password incorrect** (would throw on X509Certificate2 creation)
- **PEM to PFX conversion error** (must match key and cert)

**Check:**
- Your PFX loads without error and `HasPrivateKey` is true.
- The certificate is valid for the current date.
- The certificate is issued for client authentication (Enhanced Key Usage includes "Client Authentication").

---

## 3. **Visa CA Trust Chain**

- The Visa API server's certificate must chain up to a root CA trusted by your Windows machine.
- **Install Visa's root and intermediate CA certificates** into the Windows certificate store:
  - Trusted Root Certification Authorities (root)
  - Intermediate Certification Authorities (intermediate)

---

## 4. **Proxy/SSL Inspection**

- If you are behind a corporate proxy that does SSL inspection, mTLS will fail. Try from a home network or bypass the proxy for `*.visa.com`.

---

## 5. **File Path Issues**

- UNC paths (`\\winfs01\shared\...`) can cause permission or access issues. Try copying the PFX to a local path (e.g., `C:\Temp\vtc_keyAndCertBundle.p12`) and reference it there for testing.

---

## 6. **Code Clean-up and Best Practices**

- You are loading the certificate twice (once as `clientCert`, once as `cert`). Only load once and use that instance.
- Remove the commented-out `ServerCertificateCustomValidationCallback` unless you are debugging a trust issue (never use it in production).

---

## 7. **Full Example: Hardened and Corrected**

```csharp
using System;
using System.IO;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Security.Cryptography.X509Certificates;
using System.Web.Script.Serialization;

namespace ConsoleApp
{
    internal class Program
    {
        // ... (data models unchanged) ...

        static void Main(string[] args)
        {
            // 1. Set TLS 1.2 before anything else!
            ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;

            // 2. Configuration
            string baseUrl = Environment.GetEnvironmentVariable("VTC_BASE_URL") ?? "https://sandbox.api.visa.com/vtc";
            string pfxPath = Environment.GetEnvironmentVariable("VTC_PFX_PATH") ?? @"C:\Temp\vtc_keyAndCertBundle.p12"; // Use local path for testing
            string pfxPassword = Environment.GetEnvironmentVariable("VTC_PFX_PASSWORD") ?? "abcd";

            if (!File.Exists(pfxPath))
            {
                Console.Error.WriteLine("Client certificate file not found: " + pfxPath);
                Environment.Exit(1);
            }

            // 3. Load client certificate ONCE
            X509Certificate2 clientCert = null;
            try
            {
                clientCert = new X509Certificate2(pfxPath, pfxPassword, X509KeyStorageFlags.MachineKeySet | X509KeyStorageFlags.Exportable);
                if (!clientCert.HasPrivateKey) throw new InvalidOperationException("Client certificate has no private key.");
                if (DateTime.UtcNow < clientCert.NotBefore || DateTime.UtcNow > clientCert.NotAfter) throw new InvalidOperationException("Client certificate expired or not yet valid.");
                Console.WriteLine("Loaded client certificate: " + clientCert.Subject);
            }
            catch (Exception ex)
            {
                Console.Error.WriteLine("Failed to load client certificate: " + ex.Message);
                Environment.Exit(2);
            }

            // 4. Setup HttpClientHandler with client certificate
            var handler = new HttpClientHandler
            {
                ClientCertificateOptions = ClientCertificateOption.Manual
            };
            handler.ClientCertificates.Add(clientCert);

            // 5. (Optional) For debugging only, never in production:
            // handler.ServerCertificateCustomValidationCallback = (sender, cert, chain, sslPolicyErrors) => true;

            using (var client = new HttpClient(handler))
            {
                client.Timeout = TimeSpan.FromSeconds(30);

                string url = baseUrl.TrimEnd('/') + "/programadmin/v1/sponsors/configuration";
                var request = new HttpRequestMessage(HttpMethod.Get, url);
                request.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));

                // Basic Auth
                string username = "your-username";
                string password = "your-password";
                string authInfo = username + ":" + password;
                string encodedAuth = Convert.ToBase64String(System.Text.Encoding.ASCII.GetBytes(authInfo));
                request.Headers.Authorization = new AuthenticationHeaderValue("Basic", encodedAuth);

                Console.WriteLine("Calling: " + url);

                HttpResponseMessage response = null;
                try
                {
                    response = client.SendAsync(request).Result;
                }
                catch (Exception ex)
                {
                    Console.Error.WriteLine("HTTP request failed: " + ex.ToString());
                    Environment.Exit(3);
                }

                string raw = response.Content.ReadAsStringAsync().Result;
                Console.WriteLine("Status: " + (int)response.StatusCode + " " + response.ReasonPhrase);

                if (!response.IsSuccessStatusCode)
                {
                    Console.Error.WriteLine("Request failed:\n" + raw);
                    Environment.Exit(4);
                }

                // ... (rest of your code unchanged) ...
            }
        }
    }
}
```

---

## 8. **If You Still Get the Error**

- **Check Windows Event Viewer → Windows Logs → Application → Source: Schannel** for detailed TLS errors.
- Try running the code on a different machine/network (to rule out proxy/inspection).
- Try your PFX in Postman (Settings → Certificates) to confirm it works there.
- If you see "A call to SSPI failed" or "The client and server cannot communicate, because they do not possess a common algorithm," your Windows may lack required cipher suites or TLS 1.2 is disabled by policy.

---

## 9. **Summary Checklist**

- [x] Set `ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;` at the very top.
- [x] Use a local PFX file, not a UNC path.
- [x] Certificate has private key and is valid.
- [x] Visa CA roots/intermediates are trusted by Windows.
- [x] No SSL-inspecting proxy in the way.
- [x] No duplicate certificate loading.
- [x] No unnecessary `ServerCertificateCustomValidationCallback`.

---

If you follow these steps and still get the error, please:
- Paste the **full exception message and stack trace**.
- Check the **Windows Event Viewer** for Schannel errors and paste the details here.
- Confirm if the same PFX works in Postman.

I can then help you pinpoint the exact cause!

