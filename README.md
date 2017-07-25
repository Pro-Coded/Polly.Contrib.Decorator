# Pollynator - part of Polly.Contrib

A Code Fix for creating implementations of interfaces, using the Polly library for resilience (e.g. handling timeouts, retry logic, etc.).

Created with, and tested with, Visual Studio 2017 (should also support Visual Studio 2015 - requires testing). 

For more information regarding Polly, please refer to https://github.com/App-vNext/Polly

## Why Pollynator?

I architect 'cloud scale' solutions, and find Polly indispensable for providing resilience and graceful handling of application or infrastructure services that are either unavailable, or temporarily unresponsive due to high load. Polly allows implementation of patterns such as Circuit Breaker, to allow the application to degrade without collapsing, perhaps by making certain features unavailable (e.g. email confirmations, new registrations) while allowing the user to continue to work with the rest of the feature set.

I tend to use technologies such as DocumentDB, Redis, Elasticsearch, and wrapping every call to these client libraries would involve considerable, potentially error-prone, hand coding. I am a fan of code generation, and saw the benefit of creating a means to create 'resilient' copies of the third-party libraries (and my own libraries), through the use of Roslyn based code generation. As a result, Pollynator follows the decorator pattern, by creating a delegated implementation of an interface, passing the actual execution of the method to Polly. If interfaces change, the code can simply be regenerated, in seconds.

The vsix package can be downloaded from the Visual Studio Gallery, [here](https://marketplace.visualstudio.com/vsgallery/8f1fb8ff-2bd8-49c4-a305-83f7e25b92b9).

## Usage

1. Install the Pollynator vsix package, the Polly NuGet package, and then create a new class within your project.

2. The class should inherit from the interface that you wish to implement.

3. The class should specify relevant using statements, including Polly.Wrap, e.g. :

    	using Polly.Wrap;
    	using System;
    	using System.Threading.Tasks;

    	class TestInterfaceClass : ITestInterface
    	{

    	}

	TIP: leaving a blank line between the opening and closing curly brackets of the class declaration will enable shortening of generated type names, e.g. Task, rather than System.Threading.Task - this is a minor quirk with the usage of Roslyn to generate code.

4. Visual Studio will display an error, indicating that the interface is not implemented; the 'lightbulb' icon will appear and provide suggestions to fix this code.

	These suggestions will include two Polly specific options:

	a) Implement Interface with Polly

	b) Implement Interface with Polly, including async await

	Both these options will provide previews of the code that will be generated. Select the appropriate option, and the code will be generated for you.

## Choosing between the two options

We (myself and the creator of the Polly library, Dylan Reisenberger, along with input from experts on Stack Overflow), spent some time examining the most appropriate way to handle async and await within both the generated, wrapped methods, and the generated helper methods that call the Polly library implementation.

The TL;DR; version is that, in the majority of cases, the async await attributes need not be applied to either the generated methods or the internal helper classes; therefore, they can be 'elided' (omitted), and the behaviour will remain the same. Thus, the recommendation for most use cases is to select "Implement Interface with Polly".

In some cases, it may be desirable to include async await attributes in the generated code; therefore, I added the option to do so - "Implement Interface with Polly, including async await". 

Further information on this subject can be viewed in the closed issues list for this project.

## Working with the generated code

Pollynator will generate the required helper methods, internal private field to hold an instance of the PolicyWrap  type and the actual instance of the type that implements the interface; these are passed into a generated constructor, and can be passed using Dependency Injection. For example:

        private readonly ITestInterface _wrappedImplemention;
        private readonly PolicyWrap _polly;

        TestInterfaceClass(ITestInterface implementation, PolicyWrap polly)
        {
            _wrappedImplemention = implementation;
            _polly = polly;
        }

After the initial code generation, subsequent executions of Pollynator will NOT overwrite existing methods, properties, events, but will create them if they are missing. 

This behaviour is designed to allow full customisation of the generated code; e.g. change the default PolicyWrap to be Policy, or CircuitBreaker, etc. as desired. PolicyWrap is chosen as the default, as it may contain one or more of the other options mentioned.

The three helper methods that may be customised, are:

        private void PollyExecuteVoid(Action method)
        {
            _polly.Execute(method);
        }

        private T PollyExecute<T>(Func<T> method)
        {
            return _polly.Execute(method);
        }

        private Task<T> PollyExecuteAsync<T>(Func<Task<T>> method)
        {
            return _polly.ExecuteAsync(method);
        }

You could, for example, implement logging in these methods, if so desired.

## Example Interface, and Generated Output

For the following interface:


    	interface ITestInterface
    	{
        	    Task<T> TestMethod1Async<T>(string someParamater);

        	    Task<T> TestMethod2Async<T>(string someParamater, int anotherParameter) where T : TestConstraintClass;

        	    void TestMethod3<T>() where T : TestConstraintClass;
    	}

    	public class TestConstraintClass
    	{
    	}

Given the following class:

    	class TestInterfaceClass : ITestInterface
    	{
      
    	}

Pollynator will generate the following output:

 		class TestInterfaceClass : ITestInterface
    	{
        	    private readonly ITestInterface _wrappedImplemention;
        	    private readonly PolicyWrap _polly;

        	TestInterfaceClass(ITestInterface implementation, PolicyWrap polly)
        	{
                _wrappedImplemention = implementation;
                _polly = polly;
        	}

        	private void PollyExecuteVoid(Action method)
        	{
                _polly.Execute(method);
        	}

        	private T PollyExecute<T>(Func<T> method)
        	{
                return _polly.Execute(method);
        	}

        	private Task<T> PollyExecuteAsync<T>(Func<Task<T>> method)
        	{
                return _polly.ExecuteAsync(method);
        	}

        	public Task<T> TestMethod1Async<T>(string someParamater)
        	{
                return PollyExecute(() => _wrappedImplemention.TestMethod1Async<T>(someParamater));
        	}

        	public Task<T> TestMethod2Async<T>(string someParamater, int anotherParameter) where T : TestConstraintClass
        	{
                return PollyExecute(() => _wrappedImplemention.TestMethod2Async<T>(someParamater, anotherParameter));
        	}

        	public void TestMethod3<T>() where T : TestConstraintClass
        	{
                PollyExecuteVoid(() => _wrappedImplemention.TestMethod3<T>());
        	}
    }



## Testing the Source Code
To test the source code, clone the repo, and make sure you have Polly.Contrib.Pollynator.Vsix set as the startup project. 

Running the project will start a clean instance of Visual Studio, with the Pollynator CodeFix installed. 

You can then create a class that is intended to implement an interface, and Visual Studio will flag that this interace is not implemented.
One of the options to automatically implement the interface, is 'Implement Interface Decorated with Polly'. Refer to the above text for further information.

To help test this extension, a sample project containing classes with unimplemented interfaces (e.g. Redux, DocumentDB Client) is available [here](https://github.com/Pro-Coded/Polly.Contrib.Pollynator.Sample), for you to clone or download.






