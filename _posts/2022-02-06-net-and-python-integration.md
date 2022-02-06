---
layout: post
title: ".NET and Python integration with pythonnet"
---

It is a sprint planning. You and your colleagues are experienced in .NET development, and
pretty confident about the future sprint. The Product owner joins the meeting with a surprising new
requirement. "The .NET library you made is really solid. However, we recently
found clients, who are interested in Python version of our product. How long would you think
it take to re-write the whole thing in Python?". Such a request can stun some people - remaking
everything from ground-up is a daunting task. However, no need to rewrite your application,
if you want to re-use it on different programming language environment! Solutions exist
between the .NET and Python, which can utilize each other in different scenarios with
minimal effort.

In the following chapters we will talk you through on how to write Python code,
using .NET libraries and vice-versa. You will learn how to exchange data back and forth
from Python to .NET. As a bonus, we look at how to hook Python methods to .NET events

### Prerequisites

If you want to try the examples yourself, all the code is available in [GitHub repository].
To test the examples, Python 3.7 and .NET Core 3 are required. Besides that, the snippets in repository are
run using testing framework. It is required to install ``pytest`` on Python using the following command line:

[GitHub repository]: https://github.com/tauptk/dotnet_and_python

{% highlight console %}
pip install pytest
{% endhighlight %}

Also, do not forget to build all .NET projects, found within the repository as well.
To run .NET tests, NUnit test framework is used. The build dependencies are automatically pulled and installed by
building the .NET application once

# Using .NET with Python

For a start, let's look on how to utilize .NET standard libraries from Python environment.
For integrating .NET into Python, we will use a pip package [pythonnet]. It provides all the Python developer would need
to integrate with Common Language Runtime.

[pythonnet]: https://github.com/pythonnet/pythonnet

To get started, let's install the module with `pip`:

{% highlight console %}
pip install pythonnet
{% endhighlight %}

Of course, running .NET components requires [.NET runtime] so make sure you have it installed as well.
With that set, let's look at some of the examples on how to bridge the code between Python and .NET.

[.NET runtime]: https://dotnet.microsoft.com/download

## Loading assembly

To make use of any functionality found within the .NET standard library, we need to load it into our Python
environment. In the first repository example ``test_py_using_net``, we have a class ``MyNetLibrary`` under the namespace
called ``net_app`` in the assembly ``net_app.dll``. Let's say we want to import this class into our Python environment,
because it has a function or attribute we want to access

{% highlight c# %}
namespace net_app
{
    public class MyNetLibrary
    {
    }
}
{% endhighlight %}

To begin, we need to know, where our ``net_app.dll`` assembly file is located. Assembly will not be found by `pythonnet`
if it's not in its search path. If that's the case, we need to update `sys` module by adding a directory path,
where `.dll` is located. In this example, let's say, that our assemblies are located in relative directory
``net_libraries/netstandard2.0``

![net app location](/static/net_app_location.png)

To prepare for assembly import to our environment, let's add an assembly location directory to the search path using
`sys.path.append`. After that, let's update our Common Language Runtime `clr` with a new assembly.
Use `clr.AddReference` to include our assembly name without file extension (for file ``net_app.dll`` it would be
``net_app``).

With that set, assembly is ready to be imported like a regular Python module! Simply import using
``from <namespace> import <class>``, where ``<namespace>`` is the entity namespace within the assembly we want to access
and ``<class>`` is our class name.

{% highlight python %}
import pytest
import clr
import sys

def my_net_object():
    path_with_net_assemblies = "net_libraries/netstandard2.0"
    sys.path.append(path_with_net_assemblies)

    # Add our net_app.dll as a reference
    clr.AddReference("net_app")

    # Importing our .Net module!
    from net_app import MyNetLibrary

    # Initializing new class instance just
    # like we write Python code
    my_net_object = MyNetLibrary()
    return my_net_object
{% endhighlight %}

If import succeeds, we can simply create a new instance of an object and use it in our Python code! Continuing on, let's
look at how to pass values in .NET methods back and forth.

## Retrieve string values

Let's say, that we have a simple method on .NET side, which, in the end, returns a string value:

{% highlight c# %}
public string ReturnString()
{
    return "Hello World from .NET!";
}
{% endhighlight %}

Calling this method from Python is simple as it can get, and since the return
value is a simple type, ``pythonnet`` takes care with casting its type to native Python string type

{% highlight python %}
def test_return_string(my_net_object):
    res = my_net_object.ReturnString()
    assert res == "Hello World from .NET!"
{% endhighlight %}

## Retrieve list objects

Retrieving reference type is easy as well. In this example, a method returns a list of strings:

{% highlight c# %}
public List<string> ReturnList()
{
    return new List<string> { "A", "B", "C" };
}
{% endhighlight %}

Python casts returned value to its native format automatically
and can successfully assert with a simple Python list

{% highlight python %}
def test__get_value_from_assembly(my_net_object):
    returned_list = my_net_object.ReturnList()

    assert type(returned_list) == list
    assert returned_list == ["A", "B", "C"]
{% endhighlight %}

## Pass string values

Let's switch thing up a bit. In this case, we will be passing Python values to the .NET method.
Starting with a simple type - a string value and returning its modified version

{% highlight c# %}
public string ModifyString(string message)
{
    return String.Format("Modified in .dll. {0}", message);
}
{% endhighlight %}

String type, received from Python, is auto casted to .NET string type and returned back.
This makes simple passing back and forth simple types - no additional integration is required

{% highlight python %}
def test__passing_string_value_to_assembly(my_net_object):
    my_message = "Python String"
    returned_string = my_net_object.ModifyString(my_message)
    assert returned_string == "Modified in .dll. Python String"
{% endhighlight %}

However, things are a little bit different, when you are trying to pass reference types to .NET method from python...

## Pass list values

Is this code block, we have a .NET method, which accepts a list of strings, and modifies this collection, returning
it back to the caller

{% highlight c# %}
public List<string> ModifyList(List<string> data)
{
    data.Reverse();
    return data;
}
{% endhighlight %}

So far, we had little trouble using .NET methods to retrieve and pass values. However, if we tried to pass simple
python list to the method, we would receive the exception:

{% highlight python %}
def test__does_not_accept_py_list(my_net_object):
    my_net_object.ModifyList(["1", "2", "3"])
{% endhighlight %}

{% highlight console %}
>   my_net_object.ModifyList(["1", "2", "3"])
E   TypeError: No method matches given arguments for ModifyList
{% endhighlight %}

Unfortunately, when passing values to .NET methods, we need to cast the list to .NET native list format beforehand.
Otherwise, we will get a type mismatch.

How do we create .NET collection purely within Python?

* add a reference to ``System.Collection`` namespace. We could skip this step, however, we would get a warning, that `Implicit loading is deprecated`. To comply with the deprecation, we need to tell exactly which reference is used to import the required classes
* import ``System.Collections.Generic.List``. This will be used to create a new instance of the list
* create a new instance of the list. We will use Python ``str`` as base type. Don't worry - ``pythonnet`` will automatically translate it to ``System.String``
* populate our newly created list with members. Sadly, constructor is not supported in this case - we will need to add members one by one using ``Add``.

{% highlight python %}
def test__accept_list_value_in_assembly(my_net_object):
    # Import Systems.Collection namespace from .NET
    clr.AddReference('System.Collections')

    # Import all required classes
    from System.Collections.Generic import List

    # Create new .NET-based list, in similar way as C#:
    # var my_new_list = new List<String>();
    my_new_list = List[str]()
    # List[String] ignores constructor.
    # Here we populate elements one by one
    [my_new_list.Add(x) for x in ("1", "2", "3")]

    returned_list = my_net_object.ModifyList(my_new_list)

    assert returned_list == ["3", "2", "1"]
{% endhighlight %}

Returned value is converted to native Python list type - we will have little trouble asserting it
with other python lists

## Pass array values

How do we interact with .NET arrays? In the following simple method, we have a .NET method, which accepts and returns
modified array. One thing to note - a new instance of array will be created as a method result

{% highlight c# %}
public string[] ModifyArray(string[] data)
{
    return data.Reverse().ToArray();
}
{% endhighlight %}

Unlike with lists, ``pythonnet`` successfully passes Python list as an argument to the method. The list is automatically
translated to string array. But how about the return value? Unfortunately, assertion of returned array with
a simple python list fails - types don't match.

{% highlight python %}
def test__return_array_not_same_type_as_py_list(my_net_object):
    returned_array = my_net_object.ModifyArray(["1", "2", "3"])

    assert returned_array == ["3", "2", "1"]
{% endhighlight %}

{% highlight console %}
Expected :['3', '2', '1']
Actual   :<System.String[] object at 0x000001DDE1E83208>
{% endhighlight %}

To make a successful assertion, we need to convert returned array to Python list. Since the return value is iterable,
we can make an easy cast to python ``list()`` for comparison

{% highlight python %}
def test__accept_array_value_in_assembly(my_net_object):
    returned_array = my_net_object.ModifyArray(["1", "2", "3"])

    assert list(returned_array) == ["3", "2", "1"]
{% endhighlight %}

## Events

It is hard to image a modern .NET application without event usage.
Luckily, ``pythonet`` has support for that as well, given we
have a simple event, which can be invoked in our application:

{% highlight c# %}
public event EventHandler<string> NetEvent;
public void RaiseNetEvent(string message)
{
    NetEvent?.Invoke(this, message);
}
{% endhighlight %}

We can use same .NET syntax to bind the event to our Python
handler. Simply ``+=`` the method you wish to bind and await for
the triggered events:

{% highlight c# %}
class TestNetEvents:
    _event_result = None

    def _accept_event(self, sender, args):
        self._event_result = args

    def test__raise_net_event(self, my_net_object):
        my_net_object.NetEvent += self._accept_event
        my_net_object.RaiseNetEvent("my message")

        assert self._event_result == "my message"
{% endhighlight %}

To see a slightly more advanced example of asynchronous event, see
[test_py_using_net.test_examples.TestAsyncNetEvents] in the demo repository ``tauptk/dotnet_and_python``

[test_py_using_net.test_examples.TestAsyncNetEvents]: https://github.com/tauptk/dotnet_and_python/blob/0104d39fe267e3f5de97ca67a7895ae23ba6741e/test_py_using_net/test_examples.py#L93

## Use cases

Where .NET applications and libraries could be integrated into python code? Here are some few examples:

* **Reusing existing resources**. Sometimes you want to integrate a library, which is not available on Python. If one exists as dynamic-link library, you can easily integrate to your python code.

* **Holding sensitive logic in .DLL**. While Python code can be obfuscated (such solutions exist like [PyArmor]), .NET can compare with a larger variety of commercially available obfuscators. Tools like [SmartAssembly] and [Babel Obfuscator] provide a rich set of features for protecting your code. If you are distributing code to Python environment you don't have full control of (In [PyArmor case], you must be using same Major/Minor version), obfuscated .NET standard library serves as a great alternative to hold your secure platform-independent code while keeping "Pythonic" interface

[PyArmor]: https://github.com/dashingsoft/pyarmor
[SmartAssembly]: https://www.red-gate.com/products/dotnet-development/smartassembly/
[Babel Obfuscator]: https://www.babelfor.net/
[PyArmor case]: https://pyarmor.readthedocs.io/en/latest/understand-obfuscated-scripts.html#the-differences-of-obfuscated-scripts

# Using Python with .NET

For using Python within .NET, we will use [Python.Included] NuGet package. If you are using windows, it is the most
straightforward approach - the module has built-in Python runtime. Once we use Python runtime for the first time,
it is automatically extracted from ``Python.Included.dll`` to the local user directory and re-used for any future calls
to Python.

[Python.Included]: https://www.nuget.org/packages/Python.Included/

![nuget2](/static/nuget2.png)

If you are using other operating system, you may need to have Python environment pre-installed to the local file system
with a Nuget package: either [pythonnet nuget] or [pythonnet_netstandard].


[pythonnet nuget]: https://www.nuget.org/packages/pythonnet
[pythonnet_netstandard]: https://github.com/henon/pythonnet_netstandard

![nuget](/static/nuget.png)

Once dependencies are set, you are ready to work with Python on your .NET app!

## Understanding Locks and Scopes

To execute a Python code within .NET application, a [Global Interpreted Lock] needs to be acquired. This lock limits
.NET application to use Python resources on a single thread only. If multi-threading is something of high importance,
see [IronPython] as an alternative of ``CPython`` based ``pythonnet``

[Global Interpreted Lock]: https://wiki.python.org/moin/GlobalInterpreterLock
[IronPython]: https://ironpython.net/

``Scopes`` are like virtual environments - each code execution, import and variable is unique for the scope. Other
opened scope will not have access to any resources created in the previous one. Below is the example, how ``GIL``
and ``scope`` are used to execute simple Python code:

{% highlight c# %}
public void BasicHelloWorld()
{
    // Python.Included unique. Executed once. Sets up 
    // python environment in local users directory
    Installer.SetupPython().Wait();

    // Aquire Python global interpreted lock
    using (var pyLock = Py.GIL())
    {
        // Crete our Python scope
        using (var scope = Py.CreateScope())
        {
            // Create Python variable executing code
            // in string format
            scope.Exec("message = 'Hello World!'");

            // Aquire and print our Python variable value
            var message = scope.Get<string>("message");
            Console.WriteLine(message);
        }
    }
}
{% endhighlight %}

## Loading python module

Let's say our Python application is located in `py_app` directory.
Here, we can find our class we are going to use in examples.py file

![py_app](/static/py_app.png)

With `__init__.py` we import our `MyHelloWorldPyClass` class
and append it to ``__all__`` variable. This will make future imports of
our class much simpler just by referencing module without the file.

{% highlight python %}
# __init__.py
from .examples import MyHelloWorldPyClass

__all__ = [
    "MyHelloWorldPyClass"
]
{% endhighlight %}

As described before, to start using Python in our .NET application,
we need to initialize Python environment and acquire Python
engine lock. Once that is done, we can start using Python. While we
talked briefly before about scopes - it's not mandatory. We will be
working on the global scope. And variable type `dynamic` will help us
greatly to code with ease

In the following example, do the following:

* Initialize Python in .NET environment
* Add the current directory to search path for successful module import (`py_app` module is located under same directory)
* Import our custom Python module `py_app`
* Create new `MyHelloWorldPyClass` class instance. Since we have added it to ``__init__.py``, we can use it directly without the need to reference the file, which holds the class

{% highlight c# %}
Py.GILState _pythonEngineLock;
dynamic _helloWorldClass;

// Initialize Python environment in local file system
// If no python environment is present - it is extracted
// If already present - reuse it
Installer.SetupPython().Wait();

// Aquire Python global interpreter lock
_pythonEngineLock = Py.GIL();

// Append our python module to search path
dynamic sys = Py.Import("sys");
sys.path.append("./");

// import our python module
dynamic py_app = Py.Import("py_app");

// Initialize our Python object
_helloWorldClass = py_app.MyHelloWorldPyClass();
{% endhighlight %}

## Retrieve string values

Starting off simple, we have a python method returning a string value

{% highlight python %}
def return_string(self):
    return "Hello World from Python!"
{% endhighlight %}

If we create our Python class instance, we will be able to get our
function called just by using its name. Since our class value type
within .NET is ``dynamic``, the application has no trouble compiling even
with no straight reference to Python method

{% highlight c# %}
public void ReturnStringFromPy()
{
    dynamic result = _helloWorldClass.return_string();

    // We get PyObject returned
    Assert.AreEqual(typeof(PyObject), result.GetType());

    // Let's cast it to string to compare its value
    Assert.AreEqual((string)result, "Hello World from Python!");
}
{% endhighlight %}

Returned value of any function call for Python using ``pythonnet`` is
``PyObject``. However, it can be easily cast to other object types like with the help of ``(string)``

## Retrieve list objects

How do we work with lists, retrieved from Python?
In this example - we have a simple Python method,
returning a list of ``string`` values

{% highlight python %}
def return_list(self):
    return ["A", "B", "C"]
{% endhighlight %}

Since we retrieve ``PyObject`` using Python method, we need to cast
result value to List. The cast order is the following:

* Cast return value to ``IEnumerable``
* Cast its members to ``dynamic`` type
* Once we have ``IEnumerable<dynamic>``, we can iterate through its members and cast them to string
* Casting final result to ``List`` if required

{% highlight c# %}
public void ReturnListFromPy()
{
    dynamic returnedValue = _helloWorldClass.return_list();

    // We get PyObject returned
    Assert.AreEqual(typeof(PyObject), returnedValue.GetType());

    // Some Linq magic to cast to native .NET format
    var returnedValueAsNetList = ((IEnumerable) returnedValue)
        .Cast<dynamic>()
        .Select(x => (string)x)
        .ToList();
    Assert.AreEqual(
        returnedValueAsNetList, 
        new List<string> { "A", "B", "C"}
    );
}
{% endhighlight %}

## Pass string values

With passing values to Python methods, we have little trouble when
using ``pythonnet``. In this example - a python method, receiving
argument, which is assumed as string type

{% highlight python %}
def modify_string(self, message):
    return "%s. Modified by Python" % message
{% endhighlight %}

Through .NET, we do not need to execute any casts - string value is
casted to Python-type string automatically

{% highlight c# %}
public void ModifyStringThroughPy()
{
    var res = _helloWorldClass.modify_string("My string");
    Assert.AreEqual(
        "My string. Modified by Python", 
        (string)res
    );
}
{% endhighlight %}

## Pass list values

Does automatic casting works with more complex object types as well? Here, we have a method,
which works with an argument as it was a python list

{% highlight python %}
def modify_list(self, data):
    data.append("A")
    data.append("B")
    data.append("C")
    return data
{% endhighlight %}

Again, we do not need to make manual casting when passing commonly used
types - .NET string list is automatically casted to Python list when
using the function

{% highlight c# %}
public void ModifyListThroughPy()
{
    var list = new List<string>() {"1", "2", "3"};
    var returnedValue = _helloWorldClass.modify_list(list);

    // Cast our returned value back to enumerable format
    var returnedValueAsNetList = ((IEnumerable)returnedValue)
        .Cast<dynamic>()
        .Select(x => (string)x)
        .ToList();
    Assert.AreEqual(
        new List<string>{ "1", "2", "3" , "A", "B", "C"}, 
        returnedValueAsNetList
    );
}
{% endhighlight %}

## Use cases

What are the use cases of using Python on-top of .NET? One of the most common problems this combination solves is plugin
support - if you want to give you users the ability to extend your software - you want it to be as simple as possible.
It is totally possible to write plugins in .NET, but when it comes to customizing software behavior, the industry
seems to favor scripting languages because of simplicity. Python, together with .NET is the most popular
choice and is supported by many desktop applications. Here is a [list of existing solutions], which combine
.NET and Python technologies.

[list of existing solutions]: https://github.com/pythonnet/pythonnet/wiki/Projects-using-pythonnet

Personally, throughout the years at [Telesoftas], combining Python with .NET really helped with development planning,
where we can bring onboard people with different technological background to the project. People, specializing in .NET
or Python, can work together effortlessly on the same project, same product. If we ever feel like lacking hands on
development, people with skills in different different technologies can be brought in given the right tools.

[Telesoftas]: https://telesoftas.com/

# Conclusion

If you ever need to quickly product from Python to .NET and vice-versa, consider tools like
``pythonnet``, which minimizes the amount needed to duplicate code across the platforms.
Writing important pieces of logic once, reusing them through different interfaces across
different programming languages - a smart way for [DRY].

[DRY]: https://en.wikipedia.org/wiki/Don%27t_repeat_yourself
