# Occlum
[![All Contributors](https://img.shields.io/badge/all_contributors-7-orange.svg?style=flat-square)](CONTRIBUTORS.md)

Occlum is a *memory-safe*, *multi-process* library OS (LibOS) for [Intel SGX](https://software.intel.com/en-us/sgx). As a LibOS, it enables *legacy* applications to run on SGX with *little or even no modifications* of source code, thus protecting the confidentiality and integrity of user workloads transparently.

Occlum has the following salient features:

  * **Efficient multitasking.** Occlum offers _light-weight_ LibOS processes: they are light-weight in the sense that all LibOS processes share the same SGX enclave. Compared to the heavy-weight, per-enclave LibOS processes, Occlum's light-weight LibOS processes is up to _1,000X faster_ on startup and _3X faster_ on IPC. In addition, Occlum offers an optional _multi-domain [Software Fault Isolation](http://www.cse.psu.edu/~gxt29/papers/sfi-final.pdf) scheme_ to isolate the Occlum LibOS processes if needed.
  * **Multiple file system support.** Occlum supports various types of file systems, e.g., _read-only hashed FS_ (for integrity protection), _writable encrypted FS_ (for confidentiality protection), _untrusted host FS_ (for convenient data exchange between the LibOS and the host OS).
  * **Memory safety.** Occlum is the _first_ SGX LibOS written in a memory-safe programming language ([Rust](https://www.rust-lang.org/)). Thus, Occlum is much less likely to contain low-level, memory-safety bugs and more trustworthy to host security-critical applications.
  * **Ease-of-use.** Occlum provides user-friendly build and command-line tools. Running applications on Occlum inside SGX enclaves can be as simple as only typing several shell commands (see the next section).

## Introduction

### Hello Occlum

If you were to write an SGX Hello World project using some SGX SDK, the project would consist of hundreds of lines of code. And to do that, you have to spend a great deal of time to learn the APIs, the programming model, and the built system of the SGX SDK.

Thanks to Occlum, you can be freed from writing any extra SGX-aware code and only need to type some simple commands to protect your application with SGX transparently---in four easy steps.

**Step 1. Compile the user program with the Occlum toolchain (e.g., `occlum-gcc`)**
```
$ occlum-gcc -fPIC -pie -o hello_world hello_world.c
$ ./hello_world
Hello World
```
There are two things worth to mention. First, programs must be compiled as position-independent code (`-fPIC`) or executables (`-pie`) to be run on Occlum. Second, the Occlum toolchain is not cross-compiling, i.e., the binaries built by the Occlum toolchain is also runnable on Linux. This property makes it convenient to compile, debug, and test user programs intended for Occlum.

**Step 2. Initialize a directory as the Occlum context via `occlum init`**
```
$ mkdir occlum_context && cd occlum_context
$ occlum init
```
The `occlum init` command creates in the current working directory a new directory named `.occlum`, which contains the compile-time and run-time state of Occlum. Each Occlum context should be used for a single instance of an application; multiple applications or different instances of a single application should use different Occlum contexts.

**Step 3. Generate a secure Occlum FS image and Occlum SGX enclave via `occlum build`**
```
$ cp ../hello_world image/bin/
$ occlum build
```
The content of the `image` directory is initialized by the `occlum init` command. The structure of the `image` directory mimics that of an ordinary UNIX FS, containing directories like `/bin`, `/lib`, `/root`, `/tmp`, etc. After copying the user program `hello_world` into `image/bin/`, the `image` directory is packaged by the `occlum build` command to generate a secure Occlum FS image as well as the Occlum SGX enclave.

**Step 4. Run the user program inside an SGX enclave**
```
$ occlum run /bin/hello_world
Hello World!
```
The `occlum run` command starts up an Occlum SGX enclave, which, behind the scene, verifies and loads the associated Occlum FS image, spawns a new LibOS process to execute `/bin/hello_world`, and eventually prints the message.

### Config Occlum

Occlum can be configured easily via a config file named `Occlum.json`, which is generated by the `occlum init` command in the Occlum context directory. The user can modify `Occlum.json` to config Occlum. The default content of `Occlum.json` is
```json
{
    "vm": {
        "user_space_size": "128MB"
    },
    "process": {
        "default_stack_size": "4MB",
        "default_heap_size": "16MB",
        "default_mmap_size": "32MB"
    },
    "env": [
        "OCCLUM=yes"
    ],
    "mount": [
        {
            "target": "/",
            "type": "sefs",
            "source": "./image",
            "options": {
                "integrity_only": true
            }
        },
        {
            "target": "/root",
            "type": "sefs"
        },
        {
            "target": "/host",
            "type": "hostfs",
            "source": "."
        },
        {
            "target": "/tmp",
            "type": "ramfs"
        }
    ]
}
```
(Limitation: the `mount` key should not be modified at the moment. We will support the configuration of mount points in future version.)

## How to Use

We have built and tested Occlum on Ubuntu 16.04 with hardware SGX support. To give Occlum a quick try, one can use the Occlum Docker image by following the steps below:

Step 1-3 are to be done on the host OS (Linux):

1. Install [Intel SGX driver for Linux](https://github.com/intel/linux-sgx-driver), which is required by Intel SGX SDK.

2. Install [enable_rdfsbase kernel module](https://github.com/occlum/enable_rdfsbase), which enables Occlum to use `rdfsbase`-family instructions in enclaves.

3. Run the Occlum Docker container, which has Occlum and its demos preinstalled:
    ```
    docker run -it --device /dev/isgx occlum/occlum:0.6.0
    ```

Step 4-5 are to be done on the guest OS running inside the Docker container:

4. (Optional) Try the sample code of Intel SGX SDK to make sure that SGX is working
    ```
    cd /opt/intel/sgxsdk/SampleCode/SampleEnclave && make && ./app
    ```
5. Check out Occlum's demos preinstalled at `/root/occlum/demos`, whose README can be found [here](demos/README.md). Or you can try to build and run your own SGX-protected applications using Occlum as shown in the demos.

## How to Build and Install

To build Occlum from the latest source code, do the following steps in an Occlum Docker container (which can be prepared as shown in the last section):

1. Download the latest source code of Occlum
    ```
    mkdir occlum && cd occlum
    git clone https://github.com/occlum/occlum .
    ```
2. Prepare the submodules required by Occlum
    ```
    make submodule
    ```
3. Compile and test Occlum
    ```
    make
    make test
    ```
4. Install Occlum
    ```
    make install
    ```
   which will install the `occlum` command-line tool and other files at `/opt/occlum`.

The Occlum Dockerfile can be found at [here](tools/docker/Dockerfile). Use it to build the container directly or read it to see the dependencies of Occlum.

## What is the Implementation Status?

Occlum is being actively developed. We now focus on implementing more system calls and additional features required in the production environment.

While this project is still not mature or stable (we are halfway through reaching version 1.0.0), we have used Occlum to port many real-world applications (like Tensorflow Lite, XGBoost, GCC, Lighttpd, etc.) to SGX with little or no source code modifications. We believe that the current implementation of Occlum is already useful to many users and ready to be deployed in some use cases. 

## Why the Name?

The project name Occlum stems from the word *Occlumency* coined in Harry Porter series by J. K. Rowling. In *Harry Porter and the Order of Pheonix*, Occlumency is described as:

> The magical defence of the mind against external penetration. An obscure branch of magic, but a highly useful one... Used properly, the power of Occlumency will help shield you from access or influence.

The same thing can be said to Occlum, not for mind, but program:

> The magical defence of the program against external penetration. An obscure branch of technology, but a highly useful one... Used properly, the power of Occlum will help shield your program from access or influence.

Of course, Occlum must be run on Intel x86 CPUs with SGX support to do its magic.

## Contributors

The founders of Occlum project are
  * Hongliang Tian and Shoumeng Yan at Ant Financial; and
  * Youren Shen, Yu Chen, and Kang Chen at Tsinghua University.

This project follows the [all-contributors](https://allcontributors.org) specification. Contributions of any kind are welcome! We will publish contributing guidelines and accept pull requests after the project gets more stable.

Thanks go to [all these wonderful contributors for this project](CONTRIBUTORS.md).

## License

Occlum is released by Ant Financial under BSD License. See the copyright information [here](LICENSE).
