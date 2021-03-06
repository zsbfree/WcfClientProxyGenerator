WcfClientProxyGenerator
=========================
Utility to generate fault tolerant and retry capable dynamic proxies for WCF services based on the WCF service interface. 

With normal Service Reference or ChannelFactory instantiated clients, care must be taken to abort and recreate the client in the event that a communication fault occurs. The goal of this project is to provide an easy-to-use method of creating WCF clients that are self healing and tolerant of temporary network communication errors while still being as transparently useable as default WCF clients.

Generated proxies fully support the C# 5 async/await syntax for service contracts that define `Task` based async operations. Automatic extension of non-async ready service contracts with full async support can be created through use of the [`WcfClientProxy.CreateAsyncProxy<T>()`](README.md#async-support) factory method.

Installation
------------

    NuGet> Install-Package WcfClientProxyGenerator

Examples
--------
The following interface defines the contract for the service:

    [ServiceContract]
    public interface ITestService
    {
        [OperationContract]
        string ServiceMethod(string request);

        [OperationContract]
        Status ServiceMethod2();
    }

The proxy can then be created based on this interface by using the `Create` method of the proxy generator:

    ITestService proxy = WcfClientProxy.Create<ITestService>(c => c.SetEndpoint(binding, endpointAddress));

The proxy generated is now tolerant of faults and communication exceptions. In this example, if the first request results in a faulted channel, you would normally have to manually dispose of it. With the proxy instance, you can continue using it.

    ITestService proxy = WcfClientProxy.Create<ITestService>(c => c.SetEndpoint("testServiceConfiguration"));
    var response = proxy.ServiceMethod("request");
    var response2 = proxy.ServiceMethod("request2"); // even if the previous request resulted in a FaultException this call will still work

If there are known exceptions or responses that you would like the proxy to retry calls on, it can be configured to retry when a custom exception or response is encountered:

    var proxy = WcfClientProxy.Create<ITestService>(c =>
    {
        c.SetEndpoint("testServiceConfiguration");
        c.RetryOnException<CustomException>();
        c.RetryOnException<PossibleCustomException>(e => e.Message == "retry only for this message");
        c.RetryOnResponse<IResponseStatus>(r => r.StatusCode == 503 || r.StatusCode == 504);
    });

Responses can also be intercepted and transformed by the proxy through use of the `HandleResponse` configuration:

    var proxy = WcfClientProxy.Create<ITestService>(c =>
    {
        c.SetEndpoint("testServiceConfiguration");
        c.HandleResponse<IResponseStatus>(where: r => r.StatusCode == 500, handler: r =>
        {
            throw new Exception("InternalServerError");
        });
    });

Using this same synchronous interface, async/await calls can be made to the `ServiceMethod` operation by creating an async enabled proxy:

    IAsyncProxy<ITestService> asyncProxy = WcfClientProxy.CreateAsyncProxy<ITestService>();

Making the request asynchronously is done by using the `CallAsync` method:

    string response = await asyncProxy.CallAsync(m => m.ServiceMethod("request"));

Synchronous calls are still supported using the `IAsyncProxy<ITestService>` proxy by accessing the `Client` property:

    string response = asyncProxy.Client.ServiceMethod("request");

Usage
-----
To create a proxy, use the `WcfClientProxy.Create<TServiceInterface>()` method. There are multiple overloads that can be used to setup and configure the proxy.

#### WcfClientProxy.Create\<TServiceInterface\>()
Calling create without passing any configuration in will configure the proxy using the `endpoint` section in your config where the `contract` attribute matches `TServiceInterface`. If more than one `endpoint` section exists, an `InvalidOperationException` is thrown. The alternate overloads must be used to select the appropriate endpoint configuration.

#### WcfClientProxy.Create\<TServiceInterface\>(string endpointConfigurationName)
This is a shortcut to using the overload that accepts an `Action<IRetryingProxyConfigurator>`. It's the same as calling `WcfClientProxy.Create<TServiceInterface>(c => c.SetEndpoint(endpointConfigurationName))`.

#### WcfClientProxy.Create\<TServiceInterface\>(Action\<IRetryingProxyConfigurator\> config)
Exposes the full configuration available. See the [Configuration](#configuration) section of the documentation.

Async Support
-------------
At runtime, generating the proxy will ensure that all non-async operation contract methods have an associated Async implementation. What this means is that if your service contract is defined as:

	[ServiceContract]
	interface IService
	{
		[OperationContract]
		int MakeCall(string input);
	}

then the proxy returned by calling `WcfClientProxy.Create<IService>()` will automatically define a `Task<int> MakeCallAsync(string input)` operation contract method at runtime.
	
To utilize this runtime generated async method, an async friendly wrapped proxy can be generated by calling the `WcfClientProxy.CreateAsyncProxy<TServiceInterface>()` method. This call returns a type `IAsyncProxy<TServiceInterface>` that exposes a `CallAsync()` method.

The `int MakeCall(string input)` method can now be called asynchronously like:

    var proxy = WcfClientProxy.CreateAsyncProxy<IService>();
    int result = await proxy.CallAsync(s => s.MakeCall("test"));

It is also possible to call into the runtime generated Async methods dynamically without use of the `IAsyncProxy<TServiceInterface>` wrapper. For example, the same service contract interface with the non-async method defined can be called asynchronously as so:

	var proxy = WcfClientProxy.Create<IService>();
	int result = await ((dynamic) proxy).MakeCallAsync("test");

WCF service contract interfaces that define task based async methods at compile will automatically work with the C# 5 async/await support.

    var proxy = WcfClientProxy.Create<IServiceAsync>();
    int result = await proxy.MakeCallAsync("test");
	
### Async Limitations
Methods that define `out` or `ref` parameters are not supported when making async/await calls. Attempts to make async calls using a proxy with these parameter types will result in a runtime exception being thrown.

Configuration
-------------
When calling the `WcfClientProxy.Create<TServiceInterface>()` method, a configuration Action is used to setup the proxy. The following configuration options are available at the proxy creation time:

If no configurator is given, then a `client` configuration section with the full name of the service interface type is looked for. If no `client` configuration section is present, an `InvalidOperationException` is thrown.

#### SetEndpoint(string endpointConfigurationName)
Configures the proxy to communicate with the endpoint as configured in the _app.config_ or _web.config_ `<system.serviceModel><client>` section. The `endpointConfigurationName` value needs to match the _name_ attribute value of the `<endpoint/>`.

For example, using:

    var proxy = WcfClientProxy.Create<ITestService>(c => c.SetEndpoint("WSHttpBinding_ITestService"))

will configure the proxy based on the `<endpoint/>` as setup in the _app.config_:

    <?xml version="1.0" encoding="utf-8" ?>
    <configuration>
        <system.serviceModel>
            <client>
                <endpoint name="WSHttpBinding_ITestService"
                          address="http://localhost:23456/TestService" 
                          binding="wsHttpBinding" 
                          contract="Api.TestService.ITestService"/>
            </client>
        </system.serviceModel>
    </configuration>

#### SetEndpoint(Binding binding, EndpointAddress endpointAddress)
Configures the proxy to communicate with the endpoint using the given `binding` at the `endpointAddress`

#### HandleRequestArgument\<TArgument\>(Func\<TArgument, string, bool\> where, Action\<TArgument\> handler)
_overload:_ `HandleRequestArgument<TArgument>(Func<TArgument, string, bool> where, Func<TArgument, TArgument> handler)`

Sets up the proxy to run handlers on argument values that are used for making WCF requests. An example use case would be to inject authentication keys into all requests where an argument value matches expectation.

The `TArgument` value can be a type as specific or general as needed. For example, configuring the proxy to handle the `object` type will result in the handler being run for all operation contract arguments, whereas configuring with a sealed type will result in only those types being handled.

This example sets the `AccessKey` property for all requests inheriting from `IAuthenticatedRequest`:

	var proxy = WcfClientProxy.Create<IService>(c =>
	{
		c.HandleRequestArgument<IAuthenticatedRequest>(req =>
		{
			req.AccessKey = "...";
		});
	});

The `where` condition takes the request object as its first parameter and the actual name of the operation contract parameter secondly. This allows for conditional handling of non-specific types based on naming conventions.

For example, a service contract defined as:

	[ServiceContract]
	public interface IAuthenticatedService
	{
		[OperationContract]
		void OperationOne(string accessKey, string input);
		
		[OperationContract]
		void OperationTwo(string accessKey, string anotherInput);
	}
	
can have the `accessKey` parameter automatically filled in with the following proxy configuration:

	var proxy = WcfClientProxy.Create<IService>(c =>
	{
		c.HandleRequestArgument<string>(
			where: (req, paramName) => paramName == "accessKey",
			handler: req => "access key value");
	});
	
the proxy can now be called with with any value in the `accessKey` parameter (e.g. `proxy.OperationOne(null, "input value")` and before the request is actually started, the `accessKey` value will be swapped out with `access key value`.

#### HandleResponse\<TResponse\>(Predicate\<TResponse\> where, Action\<TResponse\> handler)
_overload:_ `HandleResponse<TResponse>(Predicate<TResponse> where, Func<TResponse, TResponse> handler)`

Sets up the proxy to allow inspection and manipulation of responses from the service. 

Similar to `HandleRequestArgument`, the `TResponse` value can be a type as specific or general as needed. For instance, `c.HandleResponse<SealedResponseType>(...)` will only handle responses of type `SealedResponseType` whereas `c.HandleResponse<object>(...)` will be fired for all responses.

For example, if sensitive information is needed to be stripped out of certain response messages, `HandleResponse` can be used to do this.

    var proxy = WcfClientProxy.Create<IService>(c =>
    {
        c.HandleResponse<SensitiveInfoResponse>(where: r => r.Password != null, handler: r =>
        {
            r.Password = null;
            return r;
        });
    });

`HandleResponse` can also be used to throw exceptions on the client side based on the inspection of responses.

Multiple calls to `HandleResponse` can be made with different variations of `TResponse` and the `Predicate<TResponse>`. With this in mind, it is possible to have more than one response handler manipulate a `TResponse` object. It is important to note that there is *no guarantee* in the order that the response handlers operate at runtime.

#### MaximumRetries(int retryCount)
Sets the maximum amount of times the the proxy will additionally attempt to call the service in the event it encounters a known retry-friendly exception or response. If retryCount is set to 0, then only one request attempt will be made.

#### RetryFailureExceptionFactory(RetryFailureExceptionFactoryDelegate)
By default, if the value configured in the `MaximumRetries` call is exceeded (i.e. a WCF call could not successfully be made after more than 1 attempt), a `WcfRetryFailedException` exception is thrown. In some cases, it makes sense to throw an exception of a different type. This method can be used to define a factory that generates the `Exception` that is thrown.

The signature of this delegate is:

    public delegate Exception RetryFailureExceptionFactoryDelegate(int retryAttempts, Exception lastException, InvokeInfo invocationInfo);

#### RetryOnException\<TException\>(Predicate\<TException\> where = null)
Configures the proxy to retry calls when it encounters arbitrary exceptions. The optional `Predicate<Exception>` can be used to refine properties of the Exception that it should retry on.

By default, the following Exception types are registered to trigger a call retry if the `MaximumRetries` count is greater than 0:

* ChannelTerminatedException
* EndpointNotFoundException
* ServerTooBusyException

#### RetryOnResponse\<TResponse\>(Predicate\<TResponse\> where)
Configures the proxy to retry calls based on conditions in the response from the service.

For example, if your response objects all inherit from a base `IResponseStatus` interface and you would like to retry calls when certain status codes are returned, the proxy can be configured as such:

    ITestService proxy = WcfClientProxy.Create<ITestService>(c =>
    {
        c.SetEndpoint("testServiceConfiguration");
        c.RetryOnResponse<IResponseStatus>(r => r.StatusCode == 503 || r.StatusCode == 504);
    });
    
The proxy will now retry calls made into the service when it detects a `503` or `504` status code.

#### SetDelayPolicy(Func\<IDelayPolicy\> policyFactory)
Configures how the proxy will handle pausing between failed calls to the WCF service. See the [Delay Policies](#delay-policies) section below.

Instances of `IDelayPolicy` are generated through the provided factory for each call made to the WCF service.

For example, to wait an exponentially growing amount of time starting at 500 milliseconds between failures:

	ITestService proxy = WcfClientProxy.Create<ITestService>(c =>
    {
    	c.SetDelayPolicy(() => new ExponentialBackoffDelayPolicy(TimeSpan.FromMilliseconds(500)));
    });

#### OnCallBegin
Event handler that is fired immediately before a service request is made.

#### OnCallSuccess
Event handler that is fired after the service request completes successfully. Returns the count of call attempts made and the overall elapsed time that the request took.

#### OnBeforeInvoke & OnAfterInvoke
Allows configuring event handlers that are called every time a method is called on the service.
Events will receive information which method was called and with what parameters in the `OnInvokeHandlerArguments` structure.

The `OnBeforeInvoke` event will fire every time a method is attempted to be called, and thus can be fired multiple times if you have a retry policy in place.
The `OnAfterInvoke` event will fire once after a successful call to a service method.

For example, to log all service calls:

````csharp
ITestService proxy = WcfClientProxy.Create<ITestService>(c =>
     c.OnBeforeInvoke += (sender, args) => {
        Console.WriteLine("{0}.{1} called with parameters: {2}",
            args.ServiceType.Name, args.InvokeInfo.MethodName,
            String.Join(", ", args.InvokeInfo.Parameters));
    };
    c.OnAfterInvoke += (sender, args) => {
        Console.WriteLine("{0}.{1} returned value: {2}",
            args.ServiceType.Name, args.InvokeInfo.MethodName,
            args.InvokeInfo.ReturnValue);
    };
});
int result = proxy.AddNumbers(3, 42);
````

Will print:

    ITestService.AddNumbers called with parameters: 3, 42
    ITestService.AddNumbers returned value: 45

#### OnException
Like [OnBeforeInvoke and OnAfterInvoke](#onbeforeinvoke--onafterinvoke), but for exceptions.
Allows configuring an event handler that is called if a service method call results in an exception,
such as a communication failure or a FaultException originating from the service.
Configuring this event handler will not affect to the exception that is thrown to user code.

For example, to log information of all exceptions that happen:
````csharp
ITestService proxy = WcfClientProxy.Create<ITestService>(c =>
     c.OnException += (sender, args) => {
        Console.WriteLine("Exception during service call to {0}.{1}: {2}",
            args.ServiceType.Name, args.InvokeInfo.MethodName,
            args.Exception);
    };
});
````

#### ChannelFactory
Allows access to WCF extensibility features from code for advanced use cases.
Can be used, for example, to add endpoint behaviors and change client credentials used to connect to services.

Delay Policies
--------------
Delay policies are classes that implement the `WcfClientProxyGenerator.Policy.IDelayPolicy` interface. There are a handful of pre-defined delay policies to use in this namespace.

If not specified, the `LinearBackoffDelayPolicy` will be used with a minimum delay of 500 milliseconds and a maximum of 10 seconds.


#### ConstantDelayPolicy
Waits a constant amount of time between call failures regardless of how many calls have failed.

#### LinearBackoffDelayPolicy
Waits a linearly increasing amount of time between call failures that grows based on how many previous calls have failed. This policy also accepts a maximum delay argument which insures the policy will never wait more than the defined maximum value.

#### ExponentialBackoffDelayPolicy
Same as the LinearBackoffDelayPolicy, but increases the amount of time between call failures exponentially.


License
-------
Apache 2.0
