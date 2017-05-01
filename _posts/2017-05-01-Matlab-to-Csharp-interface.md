---

layout: post

title: "Spectre team launches the site"

date: 2017-04-08

---

During the early development of Spectre, we have encountered an issue of integrating matlab algorithms with C# framework. All legacy code with mass spectrometry imaging functions was implemented in MATLAB environment, which is good for quick and powerful prototyping, but not for real-world solutions.

Nevertheless, matlab scripting suffers from lack of type safety and performance. Therefore it had been decided to create C# interface between matlab and framework to initially provide type safety and later assure performance by rewriting algorithms to C++ and just changing interface calls.

Since none of us had prior matlab programming experience, the task turned out to be slightly challenging. Fortunately enough, MATLAB Compiler SDK allows for porting code to C# dynamic library. The result of generation are two *.dll files. In order to use them, one needs to import one of the dlls (the _native suffixed or normal) to the project references. The dlls contain the classes serving role of interface between C# and matlab, each having methods designed to call its counterparts. Unfortunately, each function imported from the dll accept parameters and return values as object (C# base class). Therefore, the need of creating appropriate wrappers has arised.

``` Matlab
function [ partition, res_struct ] = divik( data, xy, varargin ) % Matlab signature
```
``` C#
public DivikResult Divik(IDataset dataset, DivikOptions options) // Desired C# signature
```

At first we wanted to convert this without looking into matlab’s code. However we had the vague idea about in and out variable types resulting from working with them earlier.
So at the beginning we tried writing results of algorithms in tests to see what values do they return. It gave us some idea of C# variable types that we should use. It was the easiest way to do it and it worked with the easiest algorithms.
However, some algorithms required reading matlab code to get an idea of type of data that will act as correct function parameters.
The hardest one was Divik algorithm. It had complicated in and out values. Especially, since it was a variadic function and it returned a couple of results while C# can only return a single one. It used HashMap with unknown possible keys and values for additional algorithm options.
It was necessary to use Matlab at this stage, so we had to run it there and check the results. Thankfully it made deducting the parameters way easier.

For example at the beginning our function definition looked like this:
``` C#
public object Divik(object numArgsOut, object data, object xy, params object[] varargin);
```

which works, but is not practical. The wrapper we have created around looks as follows:

``` C#
public DivikResult Divik(IDataset dataset, DivikOptions options);
```
where IDataset and DivikOptions are classes encapsulating all necessary parameters. Class DivikResult contains:
- double QualityIndex
- double[,] Centroids
- int[] Partition
- DivikResult[] Subregions
- and others

What is the purpose of performing conversion of this kind? The answer is simple. Matlab is storing whatever it can as a double[,], so it takes larger amount of memory than necessary and it’s impractical. In addition, even if you correctly give matlab algorithm variable int[] instead of double[,], it tends to behave unexpectedly and it returns incorrect results.

As we can see, the difference is significant and thanks to that, we can control exactly what we are sending to the function and what results are we receiving.
Once done, all we needed was to run a couple of tests to check whether the results are correct and are correctly parsed to our classes.

Results which are returned by Matlab functions can be easily converted to MWStructArray class. It is Matlab arguments tree. This class contain methods: “IsField” to check if argument in tree having name which was passed as a parameter exists. If the method returns true, we can use GetField to get access to tree fields using their name. We can parse this objects to their appropriate types such as int [], double[,] but also bool [,], int or double. For nested field “subregions” we use MWCellArray class and for each element in this array we recursively create our class “DivikResult”.
