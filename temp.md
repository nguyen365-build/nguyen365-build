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

