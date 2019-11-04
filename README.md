___Update 2019-11-04: As of jython 2.7.2b2, a new artifact
[org.python:jython-slim](https://search.maven.org/search?q=g:org.python%20AND%20a:jython-slim)
now exists with proper modular dependency structure, which makes this
jython-shaded project obsolete. :tada: See #7 for details.___

This project provides a rebundled version of the Jython library (specifically:
`org.python:jython-standalone`) with shaded (i.e., renamed) dependencies.

You can use it as follows:
```
<dependency>
	<groupId>org.scijava</groupId>
	<artifactId>jython-shaded</artifactId>
	<version>2.7.1.1</version>
</dependency>
```

With this approach, dependencies of Jython are renamed to have a package prefix
of `org.scijava.jython.shaded`. So all of Jython "just works" without
interfering with other consumers of JFFI or other dependencies you might have
on your classpath.

It leverages the [Maven Shade
Plugin](http://maven.apache.org/plugins/maven-shade-plugin/) to do it.

## Background

The [Jython](http://www.jython.org/) project is awesome. But it has a critical
problem: the [binary artifacts available from Maven
Central](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.python%22%20jython)
are unshaded uberjars, and hence not usable in large projects. In particular,
both Jython and JRuby depend on the [JFFI](https://github.com/jnr/jffi)
library, but at different and incompatible versions thereof.

The naive way to add a Jython dependency is:
```
<dependency>
	<groupId>org.python</groupId>
	<artifactId>jython</artifactId>
	<version>2.7.1</version>
</dependency>
```

But that artifact does not include the critical `Lib/` directory with all of
the native Python libraries necessary for many functions to work (e.g.,
`os.path.isdir`).

So another artifact is available which includes those libraries:
```
<dependency>
	<groupId>org.python</groupId>
	<artifactId>jython-standalone</artifactId>
	<version>2.7.1</version>
</dependency>
```

But both `jython` and `jython-standalone` are __uberjars__, meaning _they
bundle the dependencies of the project inside the JAR itself_. And they are
both __unshaded__, meaning _they don't rename the package prefixes of those
dependencies_. So if you need to use a different version of those dependencies
(e.g., because JRuby needs a different version of JFFI), then you are out of
luck.

### Example

Here is an example Jython script that produces an exception when attempting to
run it using Jython 2.5.3 when JRuby 1.7.12 and dependencies (including JFFI
1.2.7) are also on the classpath:

```
import os
print os.path.isdir('/')
```

The exception which occurs is:
```
Traceback (most recent call last):
	File "test.py", line 2, in <module>
	File ".../jython-standalone-2.5.3.jar/Lib/posixpath.py", line 195, in isdir
	File ".../jython-standalone-2.5.3.jar/Lib/posixpath.py", line 195, in isdir
java.lang.IncompatibleClassChangeError: Found class com.kenai.jffi.InvocationBuffer, but interface was expected
	at com.kenai.jaffl.provider.jffi.AsmRuntime.marshal(AsmRuntime.java:167)
	at org.python.posix.LibC$jaffl$0.stat$raw(Unknown Source)
	at org.python.posix.LibC$jaffl$0.stat(Unknown Source)
	at org.python.posix.BaseNativePOSIX.stat(BaseNativePOSIX.java:200)
	at org.python.posix.LazyPOSIX.stat(LazyPOSIX.java:207)
	at org.python.modules.posix.PosixModule$StatFunction.__call__(PosixModule.java:954)
	at org.python.core.PyObject.__call__(PyObject.java:391)
	at posixpath$py.isdir$17(/Applications/Science/Fiji.app/jars/jython-standalone-2.5.3.jar/Lib/posixpath.py:198)
	at posixpath$py.call_function(/Applications/Science/Fiji.app/jars/jython-standalone-2.5.3.jar/Lib/posixpath.py)
	at org.python.core.PyTableCode.call(PyTableCode.java:165)
	at org.python.core.PyBaseCode.call(PyBaseCode.java:134)
	at org.python.core.PyFunction.__call__(PyFunction.java:317)
	at org.python.pycode._pyx0.f$0(New_.py:2)
	at org.python.pycode._pyx0.call_function(New_.py)
	at org.python.core.PyTableCode.call(PyTableCode.java:165)
	at org.python.core.PyCode.call(PyCode.java:18)
	at org.python.core.Py.runCode(Py.java:1275)
  ...
```

When running with `jython-shaded` on the classpath instead of
`jython-standalone`, the above script works as expected.

## FAQ

__Q:__ Why is this structured as a multi-module project?

__A:__ Due to a bug in ASM (the library that `maven-shade-plugin` uses to
rewrite the Java bytecode with the shaded package names), many of the
`<foo>$py.class` files cannot be processed properly. Fortunately, these classes
do not actually need to be rewritten, since they themselves do not directly
reference any of Jython's dependencies. So this project works around the issue
by using the Shade plugin twice: once to build a JAR of shaded dependencies
without the `<foo>.py` classes, and a second time to mix them back in to the
final artifact.
