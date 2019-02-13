# OpenCL on Linux

OpenCL on Linux works great!
But, it involves a number of components, so setting up OpenCL on Linux can be challenging, and debugging installation issues can be particularly challenging.
This guide describes how OpenCL on Linux works, where OpenCL components on Linux may come from, and initial troubleshooting steps when OpenCL on Linux is not working as expected.

## How OpenCL on Linux Works

OpenCL on Linux usually works in one of two ways:

1. *Installable Client Drivers* (ICDs)
1. *Direct Linking*

These two methods are described in detail below.

### OpenCL Installable Client Drivers on Linux

The *Installable Client Driver* (ICD) method is common on desktop installations.
When using the *Installable Client Driver* method, applications link against an *Installable Client Driver* loader (ICD loader), instead of linking directly to a specific OpenCL implementation.
The ICD loader is responsible for enumerating all of the OpenCL implementations that are installed on the system.
The application may then choose to run on any of the OpenCL implementations on the system, or may fail gracefully if no suitable implementation is found.

![Installable Client Driver Diagram](./images/OpenCL-ICDs.png)

Note that because the application links against the ICD loader, the *Installable Client Driver* method allows an application to target many different OpenCL implementations, including those that were not even available when the application was developed.

OpenCL implementations do not need to support the *Installable Client Driver* interfaces to be conformant.
Instead, the *Installable Client Driver* interface is described by the extension `cl_khr_icd`.

#### OpenCL ICD / ICD Loader Details

The spec for the `cl_khr_icd` extension may be found [here](https://www.khronos.org/registry/OpenCL/specs/2.2/html/OpenCL_Ext.html#cl_khr_icd-opencl).

OpenCL implementations that implement OpenCL ICD interfaces will return `cl_khr_icd` in their `CL_PLATFORM_EXTENSIONS` string.

The only functions that an OpenCL implementation must export to work with the OpenCL ICD loader are `clGetExtensionFunctionAddress` and `clIcdGetPlatformIDsKHR`.

````
$ nm -D --defined-only libAnOpenCLImplementation.so
000000000002b500 T clGetExtensionFunctionAddress
0000000000026d30 T clIcdGetPlatformIDsKHR
````

There are two commonly used ICD loaders.
Both ICD loaders are open source:

1. ocl-icd: https://github.com/OCL-dev/ocl-icd
1. Khronos OpenCL-ICD-Loader: https://github.com/khronosgroup/opencl-icd-loader

You'll usually get the ICD loader from your Linux distribution, although some OpenCL implementations will distribute an ICD loader, and you can always build your own ICD loader.
It's OK to have more than one ICD loader installed on your system, but there's no reason to need more than one ICD loader either, since the ICD loader is not vendor-specific.

You will need to use an ICD loader that is at least as new as the OpenCL APIs you'd like to use, since an earlier ICD loader will not export the newer APIs.
The ICD loader is backwards compatible, however, so it's fine to use a "newer" ICD loader, even if an OpenCL implementation only supports "older" APIs.

The ICD loader is almost always named `libOpenCL.so`.
`libOpenCL.so` is most likely a symbolic link to another file, which may itself be a symbolic link to yet another file.
The reasons for this are described [here](http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html).
Note that the final ICD loader may be named `libOpenCL.so.1.2`, though the `1.2` is the ICD loader version, and is unrelated to the version of the OpenCL APIs supported by the ICD loader.

The ICD loader exports all of the OpenCL API functions.

````
$ nm -D --defined-only libOpenCL.so.1.2
00000000000191c0 T clBuildProgram
000000000003ec10 T clCloneKernel
00000000000199a0 T clCompileProgram
0000000000011c50 T clCreateBuffer
...
````

The ICD loader enumerates the OpenCL implementations that are installed on the system by looking for files in the directory `/etc/OpenCL/vendors`.
The ICD loader will open each file in this directory that ends with `.icd`, and read a single line of text from the file.
The single line of text is interpreted as the full path to the OpenCL implementation shared library, which is then loaded by the ICD loader.

### Direct Linking

The *Direct Linking* method is less common, but is occasionally used in mobile or embedded applications that target a specific hardware configuration.
When using the *Direct Linking* method, applications link against a specific OpenCL implementation, rather than a generic OpenCL loader.

![Direct Linking Diagram](./images/OpenCL-DirectLinking.png)

Note that this method only enumerates the platforms supported by the one implementation that the application linked against, regardless of the number of OpenCL implementations installed on the system.

To support *Direct Linking*, the OpenCL implementation must export all of the APIs required by the application:

````
$ nm -D --defined-only libAnOpenCLImplementation.so
000000000004bd50 T clBuildProgram
000000000007cd20 T clCloneKernel
0000000000044720 T clCompileProgram
00000000000699b0 T clCreateBuffer
....
````

### Supporting Both Models?

If an OpenCL implementation wants to support both applications that link with the ICD loader and applications that link with it directly, it may do so by exporting all of the core OpenCL API functions in addition to the functions required by the ICD loader:

````
$ nm -D --defined-only libSupportsBothModels.so
000000000004bd50 T clBuildProgram
000000000007cd20 T clCloneKernel
....
0000000000052540 T clGetExtensionFunctionAddress
....
0000000000026d30 T clIcdGetPlatformIDsKHR
....
````

## Troubleshooting OpenCL on Linux

This section describes basic troubleshooting steps that apply to all Linux distributions and all OpenCL implementations.

### Step 1: Do you have `libOpenCL.so`?

The first thing to check is to ensure you have a `libOpenCL.so` on your system, and to verify that your application knows how to find it.
The `ldd` utility (or `objdump -p`) can usually perform this check:

````
$ ldd /path/to/your/application | grep OpenCL
	libOpenCL.so.1 => /path/to/your/libOpenCL.so.1 (0x00007f9182d27000)
````

If you see a full path to `libOpenCL.so` listed, as shown above, then you have a `libOpenCL.so` on your system, and your application knows how to find it - go to Step 2.
If you see `libOpenCL.so` listed, but you see `not found` instead of a path name, then you either don't have a `libOpenCL.so` on your system, or your application can't find it.

If you don't see `libOpenCL.so` listed at all, this could be caused by several reasons, but most likely your application is dynamically loading a `libOpenCL.so` or another library that uses OpenCL.
I'd still check that `libOpenCL.so` is properly installed in this case.

#### Installing and Finding `libOpenCL.so`

This section describes possible resolutions to `libOpenCL.so` being `not found`.

First, check if a `libOpenCL.so` exists on your system.
There are many ways to do this, some which will be dependent on your particular Linux distribution or configuration, especially whether you are using an *Installable Client Drivers* or *Direct Linking*.

The `locate` or `find` commands may be helpful: `locate libOpenCL.so`.
If you are using one, your package manager may be also helpful: start by searching for packages similar to `ocl-icd`, or `OpenCL`, or `ICD` for the *Installable Client Driver* method, or for your target OpenCL implementation for the *Direct Linking* method.

If you don't have a `libOpenCL.so` then you'll need to install it.
Again, the method to do this will be dependent on your Linux Distribution or configuration.
Remember though: unless you are using the *Direct Linking* method, your `libOpenCL.so` will be the ICD loader, and not a particular OpenCL implementation!

If you have verified that a `libOpenCL.so` exists on your system, but you still see `not found` instead of a path to your `libOpenCL.so`, then the dynamic linker does not know how to find your `libOpenCL.so`.
Here are a few possible solutions to this problem:

1. You may need to update your `ldconfig` cache file.
You can check if `libOpenCL.so` is in your cache by running `ldconfig -p | grep OpenCL`.
If `libOpenCL.so` is not in your cache file, running `ldconfig` may add it, but this will require root access.
If you install `libOpenCL.so` from a package, this step will likely be done by your package manager.
1. You can use the `LD_LIBRARY_PATH` environment variable to specify the directory containing `libOpenCL.so`.
1. You can use the `LD_PRELOAD` environment variable to preload your `libOpenCL.so` (this is uncommon).

After following these steps your application should be able to run and make OpenCL API calls, such as to `clGetPlatformIDs`.

### Step 2: Do You Have An OpenCL Implementation?

This section assumes that you have a `libOpenCL.so` on your system and that your application knows how to find it.
This means that you are able to make OpenCL API calls, such as to `clGetPlatformIDs`, but you either aren't seeing any OpenCL implementations (`platforms`) on your system, or aren't seeing one of the platforms you are expecting to see.

This section primarily discusses troubleshooting issues with the *Installable Client Driver* method, since the *Direct Linking* method will be directly calling into an OpenCL implementation.
Recall that with the *Installable Client Driver* method, `libOpenCL.so` is actually an ICD loader, which is responsible for enumerating the OpenCL implementations installed on the system.

#### Step 2a: Is your OpenCL implementation in `/etc/OpenCL/vendors`?

If you aren't seeing the OpenCL implementations you are expecting to see on your system, the first thing to check is whether the OpenCL implementation is setup so the ICD loader can find it.
You can do this by examining the contents of `/etc/OpenCL/vendors`:

````
$ ls -l /etc/OpenCL/vendors/
total 8
lrwxrwxrwx 1 root root 42 Jul 30  2018 vendor.icd -> /etc/alternatives/opencl-vendor-runtime-icd
-rw-r--r-- 1 root root 63 Feb 11 22:39 stub.icd
````

Each OpenCL implementation should have its own file in this directory, with a `.icd` file extension.
The file is probably labeled according to the vendor that provided the OpenCL implementation, but it doesn't have to.
If you don't see a file corresponding to your OpenCL implementation, or it doesn't end with a `.icd` file extension, then the ICD loader will not know how to load your OpenCL implementation!

#### Step 2b: Is the file for your OpenCL implementation readable?

If you see a file corresponding to your OpenCL implementation, the next step is to ensure that it is readable.
If it is not readable by the user running the OpenCL application, then the ICD loader will not be able to open the file to find the OpenCL implementation.
It's very unlikely that an installer will setup permissions on the file incorrectly, it's a good idea to check to be sure.

#### Step 2c: Are the contents of the file for your OpenCL implementation correct?

If you see a file corresponding to your OpenCL implementation, and the file is readable, the next step is to ensure that the contents of the file are correct.
The file for each OpenCL implementation should consist of a single line describing where to find the shared library for the OpenCL implementation.
The contents of the file is passed as-is to `dlopen`.
This means that the file should contain a single line, with no trailing whitespace, including newline characters.
In most cases, the contents of the file will be a full path the shared library for an OpenCL implementation, for example:

````
$ more vendor.icd
/opt/vendor/opencl-1.2-X.Y.Z.W/lib64/libvendorocl.so
````

It's possible that the contents of the file will simply be the name of the shared library for an OpenCL implementation, for example:

````
$ more stub.icd
libOpenCLDriverStub.so
````

This can work, but you may need to take additional steps to ensure that `dlopen` can find the shared library.
These additional steps are similar to the ones for `libOpenCL.so` described above.
To summarize:

1. You may need to update your `ldconfig` cache file.
If the shared library is not in your cache file, running `ldconfig` may add it, but this will require root access.
1. You can use the `LD_LIBRARY_PATH` environment variable to specify the directory containing the shared library for the OpenCL implementation.
1. You can use the `LD_PRELOAD` environment variable to preload the shared library for the OpenCL implementation (this is uncommon).

### Using `strace` to Troubleshoot

The Linux `strace` utility can be very helpful to troubleshoot OpenCL issues.
Since this utility can produce a lot of output, you may want to redirect the output to a file, and start by troubleshooting a simple test case, such as `clinfo`.
Here is an example command to run `strace` and redirect the output to a file:

````
$ strace ./my_opencl_application 2> trace.txt
````

You can then examine the output in `trace.txt`.
Some things you may want to look for are:

1. Was `libOpenCL.so` correctly opened?
If not, you will only see lines like this one:

    ````
    open("libOpenCL.so.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
    ````

    If you eventually see a line like this one, `libOpenCL.so` was correctly opened:

    ````
    open("/usr/lib/x86_64-linux-gnu/libOpenCL.so.1", O_RDONLY|O_CLOEXEC) = 3
    ````

1. Was the ICD loader able to open and read your vendor file?
If you see a line like this one, your vendor file was correctly opened:

    ````
    open("/etc/OpenCL/vendors/vendor.icd", O_RDONLY) = 4
    ````

1. Was the ICD loader able to open the shared library for the OpenCL implementation?
If not, you will only see lines like this one.
In this example, the vendor file did not contain a full path to the OpenCL implementation:

    ````
    open("libOpenCLDriverStub.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
    ````

    If you see a line like this one, the shared library for the OpenCL implementation was correctly opened.
    In this example, the vendor file did not contain the full path to the OpenCL implementation, but the shared library for the OpenCL implementation was in `LD_LIBRARY_PATH`:

    ````
    open("/etc/OpenCL/vendors/stub.icd", O_RDONLY) = 4
    ...
    read(4, "libOpenCLDriverStub.so\n", 23) = 23
    ...
    open("/path/to/libOpenCLDriverStub.so", O_RDONLY|O_CLOEXEC) = 5
    ````

If you've gotten this far, your application was able to find the ICD loader and the ICD loader was able to enumerate and load your vendor implementation.

### Other Things To Check

This section may add additional troubleshooting steps for specific OpenCL implementations.

---
Written by Ben Ashbaugh

This work is licensed under a Creative Commons Attribution 4.0 International License;
see http://creativecommons.org/licenses/by/4.0/

OpenCL and the OpenCL logo are trademarks of Apple Inc. used by permission by Khronos.

\* Other names and brands may be claimed as the property of others.