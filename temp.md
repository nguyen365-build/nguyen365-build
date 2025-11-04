Certainly! Here’s a complete, end-to-end guide and a full C# source code sample for calling the Visa Transaction Controls “Retrieve PAN prefix (Sandbox Only)” endpoint using mutual TLS in a .NET Framework 4.7 Console Application.

This version uses .NET Framework 4.7 APIs (not .NET Core), so there are some differences in HttpClient and certificate handling. All steps are included, from certificate setup to parsing the response and extracting the PAN prefix.

---

## 1. Prerequisites

- **.NET Framework 4.7** installed.
- **Client certificate** (PFX/P12) and password from Visa Developer.
- **Visa CA root/intermediate certificates** installed in Windows certificate store.
- **API base URL** (e.g., `https://sandbox.api.visa.com/vtc`).
- **Visual Studio** (or `csc.exe` for compiling).

---

## 2. Certificate Setup

1. **Obtain your client certificate** as a `.pfx` or `.p12` file from Visa Developer.
2. **Install Visa CA root/intermediate certificates**:
   - Double-click the `.cer` files and install them into:
     - Trusted Root Certification Authorities (root)
     - Intermediate Certification Authorities (intermediate)
3. **Store your client certificate** somewhere accessible (e.g., `C:\secrets\client-cert.pfx`).

---

## 3. Create the Console Application

1. In Visual Studio:
   - File → New → Project → Console App (.NET Framework)
   - Target Framework: **.NET Framework 4.7**
2. Add a reference to **System.Net.Http** (if not already present).

---

## 4. Full Source Code

Replace the contents of `Program.cs` with the following:

```csharp
using System;
using System.IO;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Security.Cryptography.X509Certificates;
using System.Text;
using System.Web.Script.Serialization;

namespace VtcClient47
{
    class Program
    {
        // Data models for deserialization
        public class SponsorConfigurationResponse
        {
            public string receivedTimestamp { get; set; }
            public int processingTimeinMs { get; set; }
            public ResourceObj resource { get; set; }
        }

        public class ResourceObj
        {
            public string status { get; set; }
            public string[] configuredPaymentRuleTypes { get; set; }
            public string[] availablePaymentRuleTypes { get; set; }
            public CardEnrollmentCallbackSettings cardEnrollmentCallbackSettings { get; set; }
            public CardPrefix[] cardPrefixes { get; set; }
        }

        public class CardEnrollmentCallbackSettings
        {
            public bool isCallBackDisabled { get; set; }
        }

        public class CardPrefix
        {
            public string prefixStartRange { get; set; }
            public string cardType { get; set; }
            public string cardProgramIdentifier { get; set; }
            public string prefixRangeIdentifier { get; set; }
            public bool isFlexAllowed { get; set; }
            public bool isFlexReportingAllowed { get; set; }
            public bool isVTCAllowed { get; set; }
            public bool isLeadPAN { get; set; }
        }

        static void Main(string[] args)
        {
            // Configuration: set these or read from environment/args
            string baseUrl = Environment.GetEnvironmentVariable("VTC_BASE_URL") ?? "https://sandbox.api.visa.com/vtc";
            string pfxPath = Environment.GetEnvironmentVariable("VTC_PFX_PATH") ?? @"C:\secrets\client-cert.pfx";
            string pfxPassword = Environment.GetEnvironmentVariable("VTC_PFX_PASSWORD") ?? "yourPfxPassword";

            if (!File.Exists(pfxPath))
            {
                Console.Error.WriteLine("Client certificate file not found: " + pfxPath);
                Environment.Exit(1);
            }

            // Load client certificate
            X509Certificate2 clientCert = null;
            try
            {
                clientCert = new X509Certificate2(pfxPath, pfxPassword, X509KeyStorageFlags.MachineKeySet);
            }
            catch (Exception ex)
            {
                Console.Error.WriteLine("Failed to load client certificate: " + ex.Message);
                Environment.Exit(2);
            }

            // Setup HttpClientHandler with client certificate
            var handler = new HttpClientHandler();
            handler.ClientCertificates.Add(clientCert);
            handler.ServerCertificateCustomValidationCallback = (sender, cert, chain, sslPolicyErrors) => true; // For demo only! Remove in production.

            using (var client = new HttpClient(handler))
            {
                client.Timeout = TimeSpan.FromSeconds(30);

                string url = baseUrl.TrimEnd('/') + "/programadmin/v1/sponsors/configuration";
                var request = new HttpRequestMessage(HttpMethod.Get, url);
                request.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
                // If you need to add API key/secret headers, do so here.

                Console.WriteLine("Calling: " + url);

                HttpResponseMessage response = null;
                try
                {
                    response = client.SendAsync(request).Result;
                }
                catch (Exception ex)
                {
                    Console.Error.WriteLine("HTTP request failed: " + ex.Message);
                    Environment.Exit(3);
                }

                string raw = response.Content.ReadAsStringAsync().Result;
                Console.WriteLine("Status: " + (int)response.StatusCode + " " + response.ReasonPhrase);

                if (!response.IsSuccessStatusCode)
                {
                    Console.Error.WriteLine("Request failed:\n" + raw);
                    Environment.Exit(4);
                }

                // Parse JSON using JavaScriptSerializer (available in .NET Framework)
                var serializer = new JavaScriptSerializer();
                SponsorConfigurationResponse data = null;
                try
                {
                    data = serializer.Deserialize<SponsorConfigurationResponse>(raw);
                }
                catch (Exception ex)
                {
                    Console.Error.WriteLine("JSON parse error: " + ex.Message);
                    Console.Error.WriteLine(raw);
                    Environment.Exit(5);
                }

                if (data == null || data.resource == null || data.resource.cardPrefixes == null)
                {
                    Console.Error.WriteLine("No cardPrefixes found in response.");
                    Environment.Exit(6);
                }

                // Find PAN prefix with 12 digits
                var panPrefix = data.resource.cardPrefixes
                    .Where(p => string.Equals(p.prefixRangeIdentifier, "PAN", StringComparison.OrdinalIgnoreCase)
                                && !string.IsNullOrEmpty(p.prefixStartRange)
                                && p.prefixStartRange.Length == 12)
                    .Select(p => p.prefixStartRange)
                    .FirstOrDefault();

                if (string.IsNullOrEmpty(panPrefix))
                {
                    Console.Error.WriteLine("No PAN prefix with 12 digits was found.");
                    Environment.Exit(7);
                }

                Console.WriteLine("PAN Prefix (12 digits) found: " + panPrefix);

                // Save to file
                string outputPath = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "panPrefix.txt");
                File.WriteAllText(outputPath, panPrefix);
                Console.WriteLine("Saved panPrefix to: " + outputPath);
            }
        }
    }
}
```

---

## 5. How to Run

1. **Set environment variables** (optional, or hardcode in code above):
   - `VTC_BASE_URL` (e.g., `https://sandbox.api.visa.com/vtc`)
   - `VTC_PFX_PATH` (e.g., `C:\secrets\client-cert.pfx`)
   - `VTC_PFX_PASSWORD` (your PFX password)

2. **Build and run** the project in Visual Studio or with `msbuild` and `VtcClient47.exe`.

3. **Expected output**:
   - Prints the PAN prefix (12 digits) to the console.
   - Writes it to `panPrefix.txt` in the app directory.

---

## 6. Notes

- The line `handler.ServerCertificateCustomValidationCallback = ...` disables server certificate validation for demo purposes. **Remove or replace this in production!**
- If your Visa API requires additional headers (API key, secret, etc.), add them to the request.
- If you get handshake errors, double-check your certificate, password, and CA trust chain.

---

## 7. What else can I do?

- Help you add API key/secret headers if needed.
- Show how to use Windows Certificate Store instead of a PFX file.
- Add error handling, logging, or integration with your existing systems.
- Automate this as part of a scheduled task or Windows service.

Let me know if you want any of these enhancements or have a specific error to troubleshoot!

That error means the TLS handshake failed before HTTP started. For Visa’s mTLS APIs, it’s almost always one of these: missing/invalid client cert, wrong TLS protocol, untrusted Visa CA, missing private key in the cert, or a proxy/inspection issue. Do the checks below in order.

Quick fixes to try first
- Force TLS 1.2+ in .NET Framework 4.7:
  - Put this at the very start of Main (before any request):
    ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12 | SecurityProtocolType.Tls13; // Tls13 ignored on .NET 4.7 but harmless
- Load a valid client PFX that includes the private key:
  - Ensure the file truly has the private key. When you open the .pfx in certmgr.msc MMC, it should show “You have a private key that corresponds to this certificate.”
- Use the correct Visa project client certificate:
  - The cert must belong to the Visa Developer project that is enabled for the VTC API. A wrong project cert will fail the client-auth handshake.
- Make sure you’re hitting the right endpoint:
  - Sandbox: https://sandbox.api.visa.com/vtc/...
- Temporarily log SSL errors:
  - In your handler, log ex.ToString() for any HttpRequestException and show InnerException chain.

Recommended robust setup for .NET Framework 4.7
1) TLS version and cipher suites
- .NET 4.7 defaults should negotiate TLS 1.2 automatically, but set:
  ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;
- If your OS has TLS 1.2 disabled via policy, enable it or update to a modern Windows build with current cipher suites.

2) Client certificate loading
- Prefer loading the PFX with MachineKeySet when running under non-interactive users (services/agents):
  var cert = new X509Certificate2(pfxPath, pfxPassword, X509KeyStorageFlags.MachineKeySet | X509KeyStorageFlags.Exportable);
- Verify it has a private key:
  if (!cert.HasPrivateKey) throw new InvalidOperationException("Client certificate has no private key.");
- Check expiry:
  if (DateTime.UtcNow < cert.NotBefore || DateTime.UtcNow > cert.NotAfter) throw new InvalidOperationException("Client certificate expired or not yet valid.");

3) Trust chain for server verification
- Install Visa’s root and intermediate CAs into Local Computer:
  - Trusted Root Certification Authorities (root)
  - Intermediate Certification Authorities (intermediate)
- Do not disable server validation in production. If you previously set:
  handler.ServerCertificateCustomValidationCallback = (sender, cert, chain, errors) => true;
  remove that line and fix trust properly.

4) Attach the certificate to HttpClient
var handler = new HttpClientHandler
{
    ClientCertificateOptions = ClientCertificateOption.Manual
};
handler.ClientCertificates.Add(cert);
using (var http = new HttpClient(handler)) { ... }

5) Corporate proxy/SSL inspection
- If you are behind a proxy that performs SSL inspection, the mTLS handshake will fail because the proxy can’t forward your client cert to Visa. Options:
  - Bypass the proxy for *.visa.com or the specific hosts.
  - Configure HttpClient to use the proxy and set it to not intercept those domains.
  - Test from a network without SSL interception (quick validation).

6) Correct endpoint and method
- Example URL:
  var url = "https://sandbox.api.visa.com/vtc/programadmin/v1/sponsors/configuration";
- Do not send a body with GET. Only Accept: application/json is fine.

7) Convert PEM to PFX correctly (if needed)
- If you received PEM files:
  openssl pkcs12 -export -in client_cert.pem -inkey client_key.pem -out client-cert.pfx -name "visa-client"
- Ensure the client_key.pem matches the cert. A mismatched key will cause handshake failure.

8) Validate the certificate in Windows store (optional approach)
- Import the PFX into Current User → Personal → Certificates (or Local Computer if service account).
- Then pick it by thumbprint:
  var store = new X509Store(StoreName.My, StoreLocation.CurrentUser);
  store.Open(OpenFlags.ReadOnly);
  var matches = store.Certificates.Find(X509FindType.FindByThumbprint, "YOUR_THUMBPRINT_NO_SPACES", validOnly: false);
  if (matches.Count == 0) throw new Exception("Cert not found in store");
  var certFromStore = matches[0];
  handler.ClientCertificates.Add(certFromStore);

9) Add Basic Auth or other headers only after TLS is good
- Auth headers don’t affect TLS handshakes. Fix TLS first, then add:
  request.Headers.Authorization = new AuthenticationHeaderValue("Basic", base64Creds);

Minimal hardened code snippet (drop-in for your .NET 4.7 app)
- Put at the top of Main:
ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;
- Replace your certificate loading with:
var cert = new X509Certificate2(pfxPath, pfxPassword, X509KeyStorageFlags.MachineKeySet | X509KeyStorageFlags.Exportable);
if (!cert.HasPrivateKey) throw new InvalidOperationException("Client certificate lacks private key.");
var handler = new HttpClientHandler { ClientCertificateOptions = ClientCertificateOption.Manual };
handler.ClientCertificates.Add(cert);
// Remove any handler.ServerCertificateCustomValidationCallback that returns true.
using (var client = new HttpClient(handler) { Timeout = TimeSpan.FromSeconds(30) })
{
    var request = new HttpRequestMessage(HttpMethod.Get, url);
    request.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
    var response = client.SendAsync(request).Result;
    // handle response...
}

Targeted troubleshooting checklist
- A. Cert mismatch: Are you sure the PFX belongs to the same Visa developer project/CSR used for VTC? Try another known-good client cert if available.
- B. Wrong password: Check for “The specified network password is not correct” when loading cert—means bad PFX password.
- C. Missing private key: HasPrivateKey == false => rebuild PFX with the right key.
- D. Expired cert: NotAfter < now => renew/recreate in Visa Developer.
- E. Proxy: Temporarily test on a different network or bypass proxy for sandbox.api.visa.com.
- F. CA trust: Ensure server chain builds to a trusted CA; remove any blanket “accept all certs” and install proper roots.
- G. SChannel event logs: Check Windows Event Viewer → Applications and Services Logs → Microsoft → Windows → Schannel for precise handshake errors (e.g., unknown CA, bad certificate, handshake failure, no shared cipher).

If you paste the exact InnerException or the SChannel event ID/message, I can pinpoint the cause and give a precise fix. Also, tell me whether your cert came as PFX or PEMs and whether you’re behind a corporate proxy.

