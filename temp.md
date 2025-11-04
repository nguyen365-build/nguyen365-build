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

Absolutely! If your client certificate is installed in the Windows certificate store (typically in **CurrentUser\My** or **LocalMachine\My**), you should load it from the store by its **thumbprint** (recommended) or subject name, rather than from a PFX file.

Below is a robust, production-style update for your code that loads the certificate from the Windows certificate store by thumbprint.

---

## 1. Find Your Certificate's Thumbprint

- Open `certmgr.msc` (for Current User) or `mmc.exe` with Certificates snap-in (for Local Computer).
- Locate your certificate under **Personal → Certificates**.
- Double-click it, go to **Details** tab, and copy the **Thumbprint** (remove spaces when using in code).

---

## 2. Updated Code: Load Client Certificate from Store

Replace your certificate loading section with the following:

```csharp
// Set this to your certificate's thumbprint (no spaces, all uppercase or lowercase is fine)
string certThumbprint = Environment.GetEnvironmentVariable("VTC_CERT_THUMBPRINT") ?? "YOUR_CERT_THUMBPRINT_HERE";

// Use CurrentUser or LocalMachine depending on where you installed the cert
StoreLocation storeLocation = StoreLocation.CurrentUser; // or StoreLocation.LocalMachine

X509Certificate2 clientCert = null;
try
{
    using (var store = new X509Store(StoreName.My, storeLocation))
    {
        store.Open(OpenFlags.ReadOnly);
        var certs = store.Certificates.Find(
            X509FindType.FindByThumbprint,
            certThumbprint,
            validOnly: false // set to true to only allow valid (not expired) certs
        );
        if (certs.Count == 0)
        {
            Console.Error.WriteLine("Certificate with thumbprint not found in store.");
            Environment.Exit(1);
        }
        clientCert = certs[0];
        if (!clientCert.HasPrivateKey)
        {
            Console.Error.WriteLine("Certificate does not have a private key.");
            Environment.Exit(2);
        }
        Console.WriteLine("Loaded client certificate from store: " + clientCert.Subject);
    }
}
catch (Exception ex)
{
    Console.Error.WriteLine("Failed to load client certificate from store: " + ex.Message);
    Environment.Exit(3);
}
```

---

## 3. The Rest of Your Code Remains the Same

You can now remove all references to PFX path and password.  
The rest of your code (setting up `HttpClientHandler`, adding the certificate, making the request, etc.) remains unchanged:

```csharp
var handler = new HttpClientHandler
{
    ClientCertificateOptions = ClientCertificateOption.Manual
};
handler.ClientCertificates.Add(clientCert);

using (var client = new HttpClient(handler))
{
    // ... your request code ...
}
```

---

## 4. Full Example Snippet

Here’s the relevant part of your `Main` method, ready to drop in:

```csharp
static void Main(string[] args)
{
    ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;

    string baseUrl = Environment.GetEnvironmentVariable("VTC_BASE_URL") ?? "https://sandbox.api.visa.com/vtc";
    string certThumbprint = Environment.GetEnvironmentVariable("VTC_CERT_THUMBPRINT") ?? "YOUR_CERT_THUMBPRINT_HERE";
    StoreLocation storeLocation = StoreLocation.CurrentUser; // or StoreLocation.LocalMachine

    X509Certificate2 clientCert = null;
    try
    {
        using (var store = new X509Store(StoreName.My, storeLocation))
        {
            store.Open(OpenFlags.ReadOnly);
            var certs = store.Certificates.Find(
                X509FindType.FindByThumbprint,
                certThumbprint,
                validOnly: false
            );
            if (certs.Count == 0)
            {
                Console.Error.WriteLine("Certificate with thumbprint not found in store.");
                Environment.Exit(1);
            }
            clientCert = certs[0];
            if (!clientCert.HasPrivateKey)
            {
                Console.Error.WriteLine("Certificate does not have a private key.");
                Environment.Exit(2);
            }
            Console.WriteLine("Loaded client certificate from store: " + clientCert.Subject);
        }
    }
    catch (Exception ex)
    {
        Console.Error.WriteLine("Failed to load client certificate from store: " + ex.Message);
        Environment.Exit(3);
    }

    var handler = new HttpClientHandler
    {
        ClientCertificateOptions = ClientCertificateOption.Manual
    };
    handler.ClientCertificates.Add(clientCert);

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
            Environment.Exit(4);
        }

        string raw = response.Content.ReadAsStringAsync().Result;
        Console.WriteLine("Status: " + (int)response.StatusCode + " " + response.ReasonPhrase);

        if (!response.IsSuccessStatusCode)
        {
            Console.Error.WriteLine("Request failed:\n" + raw);
            Environment.Exit(5);
        }

        // ...rest of your code...
    }
}
```

---

## 5. Notes

- **Thumbprint:** Remove all spaces when copying from the certificate details.
- **StoreLocation:** Use `CurrentUser` if you installed the cert for your user, or `LocalMachine` if installed for all users (e.g., for a service).
- **No PFX password needed** when loading from the store.
- **No need to copy the certificate file** to the app directory.

---

## 6. Troubleshooting

- If you get "certificate not found," double-check the thumbprint and store location.
- If you get "does not have a private key," re-import the certificate with the private key.
- If you get SSL/TLS errors, ensure the Visa CA root/intermediate certs are trusted.

---

Let me know if you want a full, ready-to-run code file or further help with error handling or certificate store management!

