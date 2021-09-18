# Nebula CLion




> 之前卡比同学向我咨询搭建 CLion 环境，开发 Nebula 的一些问题，我做了一些工作方便利用 Docker 在本地搭建这样一个环境，相关的东西放在：https://github.com/wey-gu/nebula-dev-CLion 。



<!--more-->

Related GitHub Repo: https://github.com/wey-gu/nebula-dev-CLion

## Run Docker Env for Nebula-Graph with CLion

Build Docker Image

```bash
git clone https://github.com/wey-gu/nebula-dev-CLion.git

cd nebula-dev-CLion
docker build -t wey/nebula-dev-clion:v2.0 .
```

Run Docker Container for Nebula-Dev with CLion Integration Readiness(actually mostly Rsync & SSH).

```bash
cd <nebula-graph-repo-you-worked-on>
export DOCKER_DEFAULT_PLATFORM=linux/amd64
docker run --rm -d \
  --name nebula-dev \
  --security-opt seccomp=unconfined \
  -p 2222:22 -p 2873:873 --cap-add=ALL \
  -v $PWD:/home/nebula \
  -w /home/nebula \
  wey/nebula-dev-clion:v2.0
```

Verify cmake with SSH.

> The default password is `password`

```bash
ssh -o StrictHostKeyChecking=no root@localhost -p 2222

# in docker
cd /home/nebula
mkdir build && cd build
cmake -DENABLE_TESTING=OFF -DCMAKE_BUILD_TYPE=Release ..
```

Access container w/o SSH.

```bash
docker exec -it nebula-dev bash
mkdir -p build && cd build
cmake -DENABLE_TESTING=OFF -DCMAKE_BUILD_TYPE=Release ..
```



## Configurations in CLion

> Ref: https://www.jetbrains.com/help/clion/clion-toolchains-in-docker.html#build-and-run

**Toolchains**

- Add a remote host
  - `root@localhost:2222`
  - `password`
- Put `/opt/vesoft/toolset/cmake/bin/cmake` as CMake

![preferences_toolchains](./images/preferences_CMake.png)

**CMake**

- Toochain:
  - Select the one created in last step
- Build directory:
  - `/home/nebula/build`

![preferences_CMake](./images/preferences_CMake.png)

## The appendix

### References of CMake output:

```bash
[root@4c98e3f77ce8 build]# cmake -DENABLE_TESTING=OFF -DCMAKE_BUILD_TYPE=Release ..
>>>> Options of Nebula Graph <<<<
-- ENABLE_ASAN                     : OFF (Build with AddressSanitizer)
-- ENABLE_BUILD_STORAGE            : OFF (Whether to build storage)
-- ENABLE_CCACHE                   : ON (Use ccache to speed up compiling)
-- ENABLE_CLANG_TIDY               : OFF (Enable clang-tidy if present)
-- ENABLE_COMPRESSED_DEBUG_INFO    : ON (Compress debug info to reduce binary size)
-- ENABLE_COVERAGE                 : OFF (Build with coverage report)
-- ENABLE_FRAME_POINTER            : OFF (Build with frame pointer)
-- ENABLE_FUZZY_TESTING            : OFF (Enable Fuzzy tests)
-- ENABLE_GDB_SCRIPT_SECTION       : OFF (Add .debug_gdb_scripts section)
-- ENABLE_JEMALLOC                 : ON (Use jemalloc as memory allocator)
-- ENABLE_MODULE_FORCE_CHECKOUT    : ON (Whether checkout branch of module to same as graph.)
-- ENABLE_MODULE_UPDATE            : OFF (Automatically update module)
-- ENABLE_PACK_ONE                 : ON (Whether to package into one)
-- ENABLE_PIC                      : OFF (Build with -fPIC)
-- ENABLE_STATIC_ASAN              : OFF (Statically link against libasan)
-- ENABLE_STATIC_UBSAN             : OFF (Statically link against libubsan)
-- ENABLE_STRICT_ALIASING          : OFF (Build with -fstrict-aliasing)
-- ENABLE_TESTING                  : OFF (Build unit tests)
-- ENABLE_TSAN                     : OFF (Build with ThreadSanitizer)
-- ENABLE_UBSAN                    : OFF (Build with UndefinedBehaviourSanitizer)
-- ENABLE_VERBOSE_BISON            : OFF (Enable Bison to report state)
-- ENABLE_WERROR                   : ON (Regard warnings as errors)
-- CMAKE_BUILD_TYPE                : Release (Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel ...)
-- CMAKE_INSTALL_PREFIX            : /usr/local/nebula (Install path prefix, prepended onto install directories.)
-- CMAKE_CXX_STANDARD              : 17
-- CMAKE_CXX_COMPILER              : /opt/vesoft/toolset/clang/9.0.0/bin/c++ (CXX compiler)
-- CMAKE_CXX_COMPILER_ID           : GNU
-- NEBULA_USE_LINKER               : bfd
-- CCACHE_DIR                      : /root/.ccache
>>>> Configuring third party for 'Nebula Graph' <<<<
-- NEBULA_THIRDPARTY_ROOT          : /opt/vesoft/third-party/2.0
-- Build info of nebula third party:
Package         : Nebula Third Party
Version         : 2.0
Date            : Mon Jun 28 15:07:38 UTC 2021
glibc           : 2.17
Arch            : x86_64
Compiler        : GCC 9.2.0
C++ ABI         : 11
Vendor          : VEsoft Inc.

-- CMAKE_INCLUDE_PATH              : /opt/vesoft/third-party/2.0/include
-- CMAKE_LIBRARY_PATH              : /opt/vesoft/third-party/2.0/lib64;/opt/vesoft/third-party/2.0/lib
-- CMAKE_PROGRAM_PATH              : /opt/vesoft/third-party/2.0/bin
-- GLIBC_VERSION                   : 2.17

-- found krb5-config here /opt/vesoft/third-party/2.0/bin/krb5-config
-- Found kerberos 5 headers: /opt/vesoft/third-party/2.0/include
-- Found kerberos 5 libs:    /opt/vesoft/third-party/2.0/lib/libgssapi_krb5.a;/opt/vesoft/third-party/2.0/lib/libkrb5.a;/opt/vesoft/third-party/2.0/lib/libk5crypto.a;/opt/vesoft/third-party/2.0/lib/libcom_err.a;/opt/vesoft/third-party/2.0/lib/libkrb5support.a
>>>> Configuring third party for 'Nebula Graph' done <<<<
-- Create the pre-commit hook
-- Creating pre-commit hook done
>>>> Configuring Nebula Common <<<<
>>>> Options of Nebula Common <<<<
-- ENABLE_ASAN                     : OFF (Build with AddressSanitizer)
-- ENABLE_CCACHE                   : ON (Use ccache to speed up compiling)
-- ENABLE_CLANG_TIDY               : OFF (Enable clang-tidy if present)
-- ENABLE_COMPRESSED_DEBUG_INFO    : ON (Compress debug info to reduce binary size)
-- ENABLE_COVERAGE                 : OFF (Build with coverage report)
-- ENABLE_FRAME_POINTER            : OFF (Build with frame pointer)
-- ENABLE_FUZZY_TESTING            : OFF (Enable Fuzzy tests)
-- ENABLE_GDB_SCRIPT_SECTION       : OFF (Add .debug_gdb_scripts section)
-- ENABLE_JEMALLOC                 : ON (Use jemalloc as memory allocator)
-- ENABLE_PIC                      : OFF (Build with -fPIC)
-- ENABLE_STATIC_ASAN              : OFF (Statically link against libasan)
-- ENABLE_STATIC_UBSAN             : OFF (Statically link against libubsan)
-- ENABLE_STRICT_ALIASING          : OFF (Build with -fstrict-aliasing)
-- ENABLE_TESTING                  : OFF (Build unit tests)
-- ENABLE_TSAN                     : OFF (Build with ThreadSanitizer)
-- ENABLE_UBSAN                    : OFF (Build with UndefinedBehaviourSanitizer)
-- ENABLE_WERROR                   : ON (Regard warnings as errors)
-- Set D_GLIBCXX_USE_CXX11_ABI to 1
-- CMAKE_BUILD_TYPE                : Release (Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel ...)
-- CMAKE_INSTALL_PREFIX            : /usr/local/nebula (Install path prefix, prepended onto install directories.)
-- CMAKE_CXX_STANDARD              : 17
-- CMAKE_CXX_COMPILER              : /opt/vesoft/toolset/clang/9.0.0/bin/c++
-- CMAKE_CXX_COMPILER_ID           : GNU
-- NEBULA_USE_LINKER               : bfd
-- CCACHE_DIR                      : /root/.ccache
>>>> Configuring third party for 'Nebula Common' <<<<
-- NEBULA_THIRDPARTY_ROOT          : /opt/vesoft/third-party/2.0
-- Build info of nebula third party:
Package         : Nebula Third Party
Version         : 2.0
Date            : Mon Jun 28 15:07:38 UTC 2021
glibc           : 2.17
Arch            : x86_64
Compiler        : GCC 9.2.0
C++ ABI         : 11
Vendor          : VEsoft Inc.

-- CMAKE_INCLUDE_PATH              : /opt/vesoft/third-party/2.0/include
-- CMAKE_LIBRARY_PATH              : /opt/vesoft/third-party/2.0/lib64;/opt/vesoft/third-party/2.0/lib
-- CMAKE_PROGRAM_PATH              : /opt/vesoft/third-party/2.0/bin
-- GLIBC_VERSION                   : 2.17

-- found krb5-config here /opt/vesoft/third-party/2.0/bin/krb5-config
-- Found kerberos 5 headers: /opt/vesoft/third-party/2.0/include
-- Found kerberos 5 libs:    /opt/vesoft/third-party/2.0/lib/libgssapi_krb5.a;/opt/vesoft/third-party/2.0/lib/libkrb5.a;/opt/vesoft/third-party/2.0/lib/libk5crypto.a;/opt/vesoft/third-party/2.0/lib/libcom_err.a;/opt/vesoft/third-party/2.0/lib/libkrb5support.a
>>>> Configuring third party for 'Nebula Common' done <<<<
-- Create the pre-commit hook
-- Creating pre-commit hook done
-- Configuring done

-- Generating done
-- Build files have been written to: /home/nebula/build/modules/common
>>>> Configuring Nebula Common done <<<<
-- Configuring done
-- Generating done
-- Build files have been written to: /home/nebula/build
```

