Got it. I’ll give you end-to-end, exhaustive instructions for calling the “Retrieve PAN prefix (Sandbox Only)” endpoint in your Visa Transaction Controls setup, including mutual TLS certificate setup and a simple C# console app with complete source code to make the request, parse the response, and validate/stash the PAN prefix. I’ll mirror what your current Postman request does and add production-grade guidance.

What this endpoint does
- Method/URL: GET {{vtcUrl}}/programadmin/v1/sponsors/configuration
- Purpose: Returns configuration info including cardPrefixes. We look for a prefixRangeIdentifier == "PAN" whose prefixStartRange has 12 digits. In Postman, that value is stored as environment variable panPrefix.
- Note: The request in Postman is documented as “Sandbox only,” but the mTLS setup below covers both sandbox and non-sandbox.

Where to find it in your workspace
- Request: Retrieve PAN prefix (Sandbox Only) [request/21471377-ee7e051b-939e-4d66-9d93-b75872a18335]
- Folder: The folder containing it inside the collection
- Collection: Visa Transaction Controls API [collection/21471377-7339d9f0-b0ad-4be4-b9f6-e136807642e2]
- Active workspace: CFCU [workspace/1a5a1459-4323-4d58-be3d-13b0d580e050]

Prerequisites
- A Visa developer project for Visa Transaction Controls with mutual TLS enabled.
- Client certificate and key issued by Visa (typically a .p12 or .pfx with a password), and Visa’s root CA chain trusted locally.
- API credentials if your specific endpoint requires headers beyond mTLS (your current Postman request only sets Content-Type: application/json).
- The base URL:
  - Sandbox example: https://sandbox.api.visa.com/vtc/programadmin/v1/sponsors/configuration (adjust to your actual {{vtcUrl}} value from your environment Visa Transaction Controls [environment/21471377-34c42b17-04d6-45c4-91c2-cc0b30ddc8b9])

Certificate setup (Windows)
1) Obtain files:
- Client certificate + private key as a .p12 or .pfx file from Visa Developer (or convert PEMs to PFX).
- Password for the PFX.
- Visa root/intermediate CA certificates (usually included or downloadable from Visa Developer).

2) Install trust chain:
- Double-click the Visa CA root/intermediate .cer files and install them into Local Computer:
  - Trusted Root Certification Authorities (root)
  - Intermediate Certification Authorities (intermediate)
- Alternatively, you can skip OS install and have your program trust them via HttpClientHandler.ServerCertificateCustomValidationCallback (not recommended for production).

3) Keep the client PFX file accessible by your program (e.g., certs/client-cert.pfx) and store its password securely (e.g., environment variable, Windows Credential Manager, Azure Key Vault). Do NOT hardcode in source for production.

Converting PEM to PFX (if needed)
If you have client_cert.pem and client_key.pem:
- On Windows (with OpenSSL installed):
  openssl pkcs12 -export -in client_cert.pem -inkey client_key.pem -out client-cert.pfx -name "visa-client" -passout pass:YourStrongPfxPassword

C# project setup
1) Create a new console app (net8.0 recommended):
- dotnet new console -n VtcClient
- cd VtcClient

2) Add packages (optional but recommended):
- System.Text.Json is part of the BCL in .NET Core. No extra packages strictly required.
- If you want dotenv support, you can add a minimal loader or set environment variables.

3) Configuration options (recommended):
- Use environment variables:
  - VTC_BASE_URL: e.g., https://sandbox.api.visa.com/vtc
  - VTC_PFX_PATH: e.g., c:\secrets\client-cert.pfx
  - VTC_PFX_PASSWORD: your PFX password

Full C# sample (complete source)
Program.cs:
using System;
using System.IO;
using System.Linq;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Security.Cryptography.X509Certificates;
using System.Text.Json;
using System.Text.Json.Serialization;
using System.Threading.Tasks;

namespace VtcClient
{
    public class Program
    {
        // Data models aligned to response shape you showed
        public class SponsorConfigurationResponse
        {
            [JsonPropertyName("receivedTimestamp")]
            public string? ReceivedTimestamp { get; set; }

            [JsonPropertyName("processingTimeinMs")]
            public int? ProcessingTimeInMs { get; set; }

            [JsonPropertyName("resource")]
            public ResourceObj? Resource { get; set; }
        }

        public class ResourceObj
        {
            [JsonPropertyName("status")]
            public string? Status { get; set; }

            [JsonPropertyName("configuredPaymentRuleTypes")]
            public string[]? ConfiguredPaymentRuleTypes { get; set; }

            [JsonPropertyName("availablePaymentRuleTypes")]
            public string[]? AvailablePaymentRuleTypes { get; set; }

            [JsonPropertyName("cardEnrollmentCallbackSettings")]
            public CardEnrollmentCallbackSettings? CardEnrollmentCallbackSettings { get; set; }

            [JsonPropertyName("cardPrefixes")]
            public CardPrefix[]? CardPrefixes { get; set; }
        }

        public class CardEnrollmentCallbackSettings
        {
            [JsonPropertyName("isCallBackDisabled")]
            public bool? IsCallBackDisabled { get; set; }
        }

        public class CardPrefix
        {
            [JsonPropertyName("prefixStartRange")]
            public string? PrefixStartRange { get; set; }

            [JsonPropertyName("cardType")]
            public string? CardType { get; set; }

            [JsonPropertyName("cardProgramIdentifier")]
            public string? CardProgramIdentifier { get; set; }

            [JsonPropertyName("prefixRangeIdentifier")]
            public string? PrefixRangeIdentifier { get; set; }

            [JsonPropertyName("isFlexAllowed")]
            public bool? IsFlexAllowed { get; set; }

            [JsonPropertyName("isFlexReportingAllowed")]
            public bool? IsFlexReportingAllowed { get; set; }

            [JsonPropertyName("isVTCAllowed")]
            public bool? IsVTCAllowed { get; set; }

            [JsonPropertyName("isLeadPAN")]
            public bool? IsLeadPAN { get; set; }
        }

        public static async Task Main(string[] args)
        {
            // Read config from environment variables or command-line
            var baseUrl = GetEnvOrFallback("VTC_BASE_URL", "https://sandbox.api.visa.com/vtc");
            var pfxPath = GetEnvOrFallback("VTC_PFX_PATH", "");
            var pfxPassword = GetEnvOrFallback("VTC_PFX_PASSWORD", "");

            if (string.IsNullOrWhiteSpace(pfxPath) || !File.Exists(pfxPath))
            {
                Console.Error.WriteLine("Error: VTC_PFX_PATH is not set or file does not exist.");
                Environment.Exit(1);
            }

            // Load client certificate
            X509Certificate2 clientCert;
            try
            {
                // X509KeyStorageFlags: use MachineKeySet if service account; here CurrentUser is fine for console
                clientCert = new X509Certificate2(
                    pfxPath,
                    pfxPassword,
                    X509KeyStorageFlags.EphemeralKeySet | X509KeyStorageFlags.Exportable
                );
            }
            catch (Exception ex)
            {
                Console.Error.WriteLine($"Failed to load client certificate: {ex.Message}");
                Environment.Exit(1);
                return;
            }

            var handler = new HttpClientHandler
            {
                ClientCertificateOptions = ClientCertificateOption.Manual
            };
            handler.ClientCertificates.Add(clientCert);

            // Optional: Enforce strong TLS
            // handler.SslProtocols = System.Security.Authentication.SslProtocols.Tls12 | System.Security.Authentication.SslProtocols.Tls13;

            // Optional: pin or validate server certificate chain more strictly
            // handler.ServerCertificateCustomValidationCallback = (message, cert, chain, errors) => {
            //     return errors == System.Net.Security.SslPolicyErrors.None;
            // };

            using var http = new HttpClient(handler)
            {
                Timeout = TimeSpan.FromSeconds(30)
            };

            // Construct endpoint URL
            var url = $"{baseUrl.TrimEnd('/')}/programadmin/v1/sponsors/configuration";

            using var req = new HttpRequestMessage(HttpMethod.Get, url);
            req.Headers.Accept.Clear();
            req.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));

            // If the API requires auth headers (e.g., x-client-id/x-client-secret, API key)
            // add them here. Your current Postman request only sets Content-Type.
            // Example:
            // req.Headers.Add("x-client-id", Environment.GetEnvironmentVariable("VISA_CLIENT_ID"));
            // req.Headers.Add("x-client-secret", Environment.GetEnvironmentVariable("VISA_CLIENT_SECRET"));

            // In Postman, Content-Type was set even for GET; typically unnecessary but harmless:
            req.Content = null; // ensure no body
            // If you must send Content-Type with GET, use req.Headers.TryAddWithoutValidation("Content-Type","application/json");

            Console.WriteLine($"Calling: {url}");
            HttpResponseMessage resp;
            try
            {
                resp = await http.SendAsync(req);
            }
            catch (HttpRequestException hre)
            {
                Console.Error.WriteLine($"Network/TLS error: {hre.Message}");
                Console.Error.WriteLine("Check certificate, base URL, and system trust store.");
                Environment.Exit(2);
                return;
            }

            var raw = await resp.Content.ReadAsStringAsync();
            Console.WriteLine($"Status: {(int)resp.StatusCode} {resp.ReasonPhrase}");
            // For troubleshooting uncomment:
            // Console.WriteLine("Raw response:");
            // Console.WriteLine(raw);

            if (!resp.IsSuccessStatusCode)
            {
                Console.Error.WriteLine("Request failed.");
                Console.Error.WriteLine(raw);
                Environment.Exit(3);
            }

            SponsorConfigurationResponse? data;
            try
            {
                data = JsonSerializer.Deserialize<SponsorConfigurationResponse>(raw, new JsonSerializerOptions
                {
                    PropertyNameCaseInsensitive = true
                });
            }
            catch (Exception ex)
            {
                Console.Error.WriteLine($"JSON parse error: {ex.Message}");
                Console.Error.WriteLine(raw);
                Environment.Exit(4);
                return;
            }

            if (data?.Resource?.CardPrefixes == null || data.Resource.CardPrefixes.Length == 0)
            {
                Console.Error.WriteLine("No cardPrefixes found in response.");
                Environment.Exit(5);
            }

            // Find a PAN prefix with 12 digits (mirrors your Postman test)
            var panPrefix = data.Resource.CardPrefixes
                .FirstOrDefault(p =>
                    string.Equals(p.PrefixRangeIdentifier, "PAN", StringComparison.OrdinalIgnoreCase) &&
                    !string.IsNullOrEmpty(p.PrefixStartRange) &&
                    p.PrefixStartRange!.Length == 12
                )?.PrefixStartRange;

            if (string.IsNullOrEmpty(panPrefix))
            {
                Console.Error.WriteLine("No PAN prefix with 12 digits was found.");
                Environment.Exit(6);
            }

            Console.WriteLine($"PAN Prefix (12 digits) found: {panPrefix}");

            // If you want to persist like Postman environment variable, write to a file or env:
            var outputPath = Path.Combine(AppContext.BaseDirectory, "panPrefix.txt");
            await File.WriteAllTextAsync(outputPath, panPrefix);
            Console.WriteLine($"Saved panPrefix to: {outputPath}");
        }

        private static string GetEnvOrFallback(string key, string fallback)
        {
            var v = Environment.GetEnvironmentVariable(key);
            return string.IsNullOrWhiteSpace(v) ? fallback : v;
        }
    }
}

How to run the sample
1) Set environment variables:
- On Windows PowerShell:
  $env:VTC_BASE_URL = "https://sandbox.api.visa.com/vtc"
  $env:VTC_PFX_PATH = "C:\secrets\client-cert.pfx"
  $env:VTC_PFX_PASSWORD = "yourPfxPassword"

2) Build and run:
- dotnet build
- dotnet run

3) Expected behavior:
- The program makes a GET call using mutual TLS.
- Parses JSON, finds a PAN prefix with 12 digits, prints it, and writes panPrefix.txt next to the executable.
- If the server responds like your Postman example, you should see 448645147895.

Common pitfalls and fixes
- Handshake failure (403/SSL/TLS error): Ensure the client PFX is the correct one for your Visa project and not expired; verify the PFX password; ensure Visa CA roots are trusted.
- 401/403 (non-TLS): If your project also enforces API key/secret headers or Basic Auth, add those headers. Check your Visa Developer project settings.
- Wrong base URL: Match {{vtcUrl}} from your active environment Visa Transaction Controls [environment/21471377-34c42b17-04d6-45c4-91c2-cc0b30ddc8b9].
- Proxy interference: If corporate proxy intercepts SSL, you may need to configure HttpClientHandler.Proxy or bypass proxy for *.visa.com.
- GET with Content-Type: Usually unnecessary; this sample omits it.

Mapping to your Postman test
- Your request validates HTTP 200 and extracts a cardPrefixes item meeting:
  - prefixRangeIdentifier == "PAN"
  - prefixStartRange length == 12
- The C# sample mirrors that logic and persists the value.

What I can do next for you
- Add code to export panPrefix to your Postman environment automatically via the Postman API or write a subsequent request that consumes this value.
- Create a small collection and monitor that calls this endpoint on a schedule and alerts on failures.
- Generate a variant of this client that uses HttpClientFactory, Polly retries, and structured logging.
- If you want, I can add a follow-up request in your collection that reads the environment variable panPrefix and continues your workflow.

If you’d like me to wire any of those into your existing collection or environment now, say the word and I’ll set it up directly in your workspace.

