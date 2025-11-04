Absolutely! You can search for a certificate in the Windows certificate store by its **Friendly Name** using the `X509Certificate2.FriendlyName` property.  
However, the `X509Store.Certificates.Find` method does **not** support searching by friendly name directly, so you need to enumerate the certificates and match the `FriendlyName` property yourself.

Below is a robust code snippet to find a certificate by its friendly name:

---

## Example: Find Certificate by Friendly Name

```csharp
using System;
using System.Linq;
using System.Security.Cryptography.X509Certificates;

class Program
{
    static void Main(string[] args)
    {
        string friendlyName = Environment.GetEnvironmentVariable("VTC_CERT_FRIENDLYNAME") ?? "YOUR_FRIENDLY_NAME";
        StoreLocation storeLocation = StoreLocation.CurrentUser; // Or StoreLocation.LocalMachine

        X509Certificate2 clientCert = null;
        try
        {
            using (var store = new X509Store(StoreName.My, storeLocation))
            {
                store.Open(OpenFlags.ReadOnly);

                // Enumerate and match by FriendlyName (case-insensitive)
                clientCert = store.Certificates
                    .OfType<X509Certificate2>()
                    .FirstOrDefault(cert =>
                        cert.FriendlyName != null &&
                        cert.FriendlyName.Equals(friendlyName, StringComparison.OrdinalIgnoreCase)
                    );

                if (clientCert == null)
                {
                    Console.Error.WriteLine("Certificate with friendly name not found in store.");
                    Environment.Exit(1);
                }
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

        // ... use clientCert as needed ...
    }
}
```

---

## Notes

- **FriendlyName** is not guaranteed to be unique. If multiple certificates have the same friendly name, this will pick the first one found.
- If you want to filter further (e.g., by validity period, issuer, etc.), you can add more conditions to the `FirstOrDefault` lambda.
- Use `StoreLocation.CurrentUser` or `StoreLocation.LocalMachine` as appropriate for where your certificate is installed.

---

## How to Set Friendly Name

If you need to set or check the friendly name:
- Open `certmgr.msc` or `mmc.exe` with the Certificates snap-in.
- Find your certificate under Personal → Certificates.
- Right-click → Properties → set the Friendly Name.

---

Let me know if you want a full working example with the HTTP request included!

