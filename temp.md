Of course. For a WCF service (`*.svc`), you can enforce HTTPS and restrict access by IP address by combining configuration in your `web.config` file with a custom piece of code that inspects incoming requests.

Here’s a complete guide on how to do it.

-----

### \#\# 1. Enforce HTTPS via Configuration 🔒

The easiest and most reliable way to ensure all traffic is over HTTPS is by configuring the service's **binding**. This tells WCF to only accept connections that are secured using transport-level security (SSL/TLS).

In your `web.config` file, find the `<system.serviceModel>` section. You'll need to define a binding that enforces transport security and then apply it to your service's endpoint.

```xml
<configuration>
  <system.serviceModel>
    <bindings>
      <wsHttpBinding>
        <binding name="HttpsOnlyBinding">
          <security mode="Transport">
            <transport clientCredentialType="None" />
          </security>
        </binding>
      </wsHttpBinding>
    </bindings>

    <services>
      <service name="YourNamespace.YourService" behaviorConfiguration="DefaultServiceBehavior">
        <endpoint address=""
                  binding="wsHttpBinding"
                  bindingConfiguration="HttpsOnlyBinding"
                  contract="YourNamespace.IYourService" />
      </service>
    </services>

    <behaviors>
      <serviceBehaviors>
        <behavior name="DefaultServiceBehavior">
          <serviceMetadata httpsGetEnabled="true" httpGetEnabled="false" />
          <serviceDebug includeExceptionDetailInFaults="false" />
        </behavior>
      </serviceBehaviors>
    </behaviors>
  </system.serviceModel>
</configuration>
```

**Key Changes:**

  * **`<security mode="Transport">`**: This is the critical line. It mandates that the connection must use transport-level security, which for `wsHttpBinding` means HTTPS.
  * **`bindingConfiguration="HttpsOnlyBinding"`**: This links your service's endpoint to the secure binding configuration you just created.
  * **`httpsGetEnabled="true"`**: Allows clients to retrieve the service metadata (WSDL) over HTTPS, which is useful for generating client proxies.

-----

### \#\# 2. Filter by IP Address using a Custom Inspector 📝

WCF doesn't have a built-in `web.config` setting for IP whitelisting. The standard approach is to create a **Message Inspector**, which is a component that can examine (and reject) messages before they reach your service operation.

#### Step A: Create the IP Address Message Inspector

This class will contain the core logic to check the client's IP address against an allowed list.

```csharp
using System;
using System.Configuration;
using System.Linq;
using System.ServiceModel;
using System.ServiceModel.Channels;
using System.ServiceModel.Dispatcher;
using System.Net; // Required for HttpStatusCode

public class IpAddressMessageInspector : IDispatchMessageInspector
{
    public object AfterReceiveRequest(ref Message request, IClientChannel channel, InstanceContext instanceContext)
    {
        // 1. Get the property that contains the remote endpoint's information.
        var prop = (RemoteEndpointMessageProperty)request.Properties[RemoteEndpointMessageProperty.Name];
        string clientIp = prop.Address;

        // 2. Get the list of allowed IPs from web.config
        string allowedIpsAppSetting = ConfigurationManager.AppSettings["AllowedIPs"];
        if (string.IsNullOrEmpty(allowedIpsAppSetting))
        {
            // If no IPs are configured, deny all for security.
            throw new WebFaultException(HttpStatusCode.Forbidden);
        }

        var allowedIps = allowedIpsAppSetting.Split(',').Select(ip => ip.Trim()).ToList();

        // 3. Check if the client's IP is in the list.
        if (!allowedIps.Contains(clientIp))
        {
            // If not allowed, throw an exception to reject the request.
            // Using WebFaultException gives a clean HTTP 403 Forbidden response.
            throw new WebFaultException(HttpStatusCode.Forbidden);
        }

        return null; // Request is allowed, continue processing.
    }

    public void BeforeSendReply(ref Message reply, object correlationState)
    {
        // No action needed before sending a reply.
    }
}
```

#### Step B: Create a Service Behavior to Apply the Inspector

You need a "behavior" to attach your custom message inspector to the WCF runtime.

```csharp
using System.ServiceModel.Description;
using System.ServiceModel.Dispatcher;

public class IpAddressFilterBehavior : IServiceBehavior
{
    public void ApplyDispatchBehavior(ServiceDescription serviceDescription, ServiceHostBase serviceHostBase)
    {
        // Add our custom message inspector to every endpoint dispatcher.
        foreach (ChannelDispatcher cd in serviceHostBase.ChannelDispatchers)
        {
            foreach (EndpointDispatcher ed in cd.Endpoints)
            {
                ed.DispatchRuntime.MessageInspectors.Add(new IpAddressMessageInspector());
            }
        }
    }

    public void AddBindingParameters(ServiceDescription serviceDescription, ServiceHostBase serviceHostBase, System.Collections.ObjectModel.Collection<ServiceEndpoint> endpoints, System.ServiceModel.Channels.BindingParameterCollection bindingParameters) { }
    public void Validate(ServiceDescription serviceDescription, ServiceHostBase serviceHostBase) { }
}
```

#### Step C: Create a Behavior Extension Element

This final piece of code makes it possible to add your new behavior directly in the `web.config` file, which is much cleaner than applying it programmatically.

```csharp
using System;
using System.ServiceModel.Configuration;

public class IpAddressFilterBehaviorExtensionElement : BehaviorExtensionElement
{
    // The type of behavior this extension creates.
    public override Type BehaviorType => typeof(IpAddressFilterBehavior);

    // Creates the behavior instance.
    protected override object CreateBehavior() => new IpAddressFilterBehavior();
}
```

-----

### \#\# 3. Update Web.config to Use the IP Filter ⚙️

Now, tie it all together in your `web.config`.

1.  **Add your allowed IPs** to the `<appSettings>` section.
2.  **Register your behavior extension**.
3.  **Apply the behavior** to your service.

<!-- end list -->

```xml
<configuration>
  <appSettings>
    <add key="AllowedIPs" value="192.168.1.100, 203.0.113.54, ::1" />
  </appSettings>
  
  <system.serviceModel>
    <bindings> ... </bindings>
    
    <services>
      <service name="YourNamespace.YourService" behaviorConfiguration="SecureAndFilteredBehavior">
        <endpoint address=""
                  binding="wsHttpBinding"
                  bindingConfiguration="HttpsOnlyBinding"
                  contract="YourNamespace.IYourService" />
      </service>
    </services>
    
    <behaviors>
      <serviceBehaviors>
        <behavior name="SecureAndFilteredBehavior">
          <serviceMetadata httpsGetEnabled="true" httpGetEnabled="false" />
          <serviceDebug includeExceptionDetailInFaults="false" />
          <ipAddressFilter />
        </behavior>
      </serviceBehaviors>
    </behaviors>
    
    <extensions>
      <behaviorExtensions>
        <add name="ipAddressFilter" 
             type="YourNamespace.IpAddressFilterBehaviorExtensionElement, YourAssemblyName" />
      </behaviorExtensions>
    </extensions>
  </system.serviceModel>
</configuration>
```

**Key Changes:**

  * **`<appSettings>`**: A new key `AllowedIPs` holds your IP whitelist.
  * **`behaviorConfiguration="SecureAndFilteredBehavior"`**: Your service now points to a new behavior configuration.
  * **`<ipAddressFilter />`**: This line adds your custom IP filtering behavior. The name `ipAddressFilter` matches the `name` you gave it in the `<extensions>` section.
  * **`<extensions>`**: This section registers your `IpAddressFilterBehaviorExtensionElement` so WCF knows what `<ipAddressFilter />` means. **Remember to replace** `YourNamespace` and `YourAssemblyName` with the actual namespace and assembly name of your project.

With these changes, any request to your service will first be required to use HTTPS. Then, the custom message inspector will run, check the client's IP against the list in `web.config`, and reject any unauthorized requests with an `HTTP 403 Forbidden` error.
