# Frequently Asked Questions

### I'm having trouble compiling `<project name here>`

First, make sure that you can compile that project natively on whatever platform you're attempting to compile it on.  Once you are assured of that, search around the internet to see if anyone else has run into issues cross-compiling that project for that platform.  In particular, most smaller projects should be just fine, but larger projects (and especially anything that does any kind of bootstrapping) may need some extra smarts smacked into their build system to support cross-compiling.  Finally, if you're still stuck, try reaching out for help on the [`#binarybuilder` channel](https://julialang.slack.com/archives/C674ELDNX) in the JuliaLang slack.

### How do I use this to compile my Julia code?

This package does not compile Julia code; it compiles C/C++/Fortran dependencies.  Think about that time you wanted to use `IJulia` and you needed to download/install `libnettle`.  The purpose of this package is to make generated tarballs that can be downloaded/installed painlessly as possible.

### What is this I hear about the macOS SDK license agreement?

Apple restricts distribution and usage of the macOS SDK, a necessary component to build software for macOS targets.  Please read the [Apple and Xcode SDK agreement](https://images.apple.com/legal/sla/docs/xcode.pdf) for more information on the restrictions and legal terms you agree to when using the SDK to build software for Apple operating systems. Copyright law is a complex area and you should not take legal advice from FAQs on the internet. This toolkit is designed to primarily run on Linux, though it can of course be used within a virtualized environment on a macOS machine or directly by running Linux Apple hardware. The Docker runner implements the virtualization approach on macOS machines.  `BinaryBuilder.jl`, by default, will not automatically download or use the macOS SDK on non-apple host operating systems, unless the `BINARYBUILDER_AUTOMATIC_APPLE` environment variable is set to `true`.

### Are there other environment variables I can use?

Yes, [take a look](environment_variables.md).

### Hey, this is cool, can I use this for my non-Julia related project?

Absolutely!  There's nothing Julia-specific about the binaries generated by the cross-compilers used by `BinaryBuilder.jl`.  Although the best interface for interacting with this software will always be the Julia interface defined within this package, you are free to use these software tools for other projects as well.  Note that the cross-compiler image is built through a multistage bootstrapping process, [see this repository for more information](https://github.com/JuliaPackaging/Yggdrasil).  Further note the **macOS SDK license agreement** tidbit above.

### At line XXX, ABORTED (Operation not permitted)!

Some linux distributions have a bug in their `overlayfs` implementation that prevents us from mounting overlay filesystems within user namespaces.  See [this Ubuntu kernel bug report](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1531747) for a description of the situation and how Ubuntu has patched it in their kernels.  To work around this, you can launch `BinaryBuilder.jl` in "privileged container" mode.  BinaryBuilder should auto-detect this situation, however if the autodetection is not working or you want to silence the warning, you can set the `BINARYBUILDER_RUNNER` environment variable to `privileged`.  Unfortunately, this involves running `sudo` every time you launch into a BinaryBuilder session, but on the other hand, this successfully works around the issue on distributions such as Arch linux.

### I have to build a very small project without a Makefile, what do I have to do?

What BinaryBuilder needs is to find the relevant file (shared libraries, or executables, etc...) organised under the `$prefix` directory: libraries should go to `${libdir}`, executables to `${bindir}`.  You may need to create those directories.  You are free to choose whether to create a simple Makefile to build the project or to do everything within the `build_tarballs.jl` script.
When the script completes, BinaryBuilder expects to find at least one artifact _built for the expected architecture_ in either `${libdir}` or `${bindir}`.
Remember also that you should use the standard environment variables like `CC`, `CXX`, `CFLAGS`, `LDFLAGS` as appropriate in order to cross compile.  See the list of variables in the [Tips for Building Packages](build_tips.md) section.

### Can I open a shell in a particular build environment for doing some quick tests?

Yes!  You can use [`BinaryBuilder.runshell(platform)`](@ref BinaryBuilderBase.runshell) to quickly start a shell in the current directory, without having to set up a working `build_tarballs.jl` script.  For example,
```
julia -e 'using BinaryBuilder; BinaryBuilder.runshell(Platform("i686", "windows"))'
```
will open a shell in a Windows 32-bit build environment, without any source loaded.  The current working directory of your system will be mounted on `${WORKSPACE}` within this BinaryBuilder environment.

### Can I publish a JLL package locally without going through Yggdrasil?

You can always build a JLL package on your machine with the `--deploy` flag to the `build_tarballs.jl` script.  Read the help (`--help`) for more information.

A common use case is that you want to build a JLL package for, say, `Libfoo`, that will be used as dependency to build `Quxlib`, and you want to make sure that building both `Libfoo` and `Quxlib` will work before submitting all the pull requests to [Yggdrasil](https://github.com/JuliaPackaging/Yggdrasil/).  You can prepare the `build_tarballs.jl` script for `Libfoo` and then build and deploy it with

```
julia build_tarballs.jl --debug --verbose --deploy="MY_USERNAME/Libfoo_jll.jl"
```

replacing `MY_USERNAME` with your GitHub username: this will build the tarballs for all the platforms requested and upload them to a release of the `MY_USERNAME/Libfoo_jll.jl`, where the JLL package will also be created.  As explained above, you can pass argument the list of triplets of the platforms for you which you want to build the tarballs, in case you want to compile only some of them.  In the Julia REPL, you can install this package as any unregistered package with

```julia
]add https://github.com/MY_USERNAME/Libfoo_jll.jl.git
```

or develop it with

```julia
]dev https://github.com/MY_USERNAME/Libfoo_jll.jl.git
```

Since this package is unregistered, you have to use the full [`PackageSpec`](https://julialang.github.io/Pkg.jl/v1/api/#Pkg.PackageSpec) specification to add it as dependency of the local builder for `Quxlib`:

```julia
    Dependency(PackageSpec(; name = "Libfoo_jll",  uuid = "...", url = "https://github.com/MY_USERNAME/Libfoo_jll.jl.git"))
```

You can of course in turn build and deploy this package with

```
julia build_tarballs.jl --debug --verbose --deploy="MY_USERNAME/Quxlib_jll.jl"
```

Note that `PackageSpec` can also point to a local path: e.g., `PackageSpec(; name="Libfoo_jll", uuid="...", path="/home/myname/.julia/dev/Libfoo_jll")`.  This is particularly useful when [Building a custom JLL package locally](@ref), instead of deploying it to a remote Git repository.

### What are those numbers in the list of sources?  How do I get them?

The list of sources is a vector of [`BinaryBuilder.AbstractSource`](@ref)s.  What the hash is depends on what the source is:

* For a [`FileSource`](@ref) or an [`ArchiveSource`](@ref), the hash is a 64-character SHA256 checksum.  If you have a copy of that file, you can compute the hash in Julia with
  ```julia
  using SHA
  open(path_to_the_file, "r") do f
       bytes2hex(sha256(f))
  end
  ```
  where `path_to_the_file` is a string with the path to the file.  Alternatively, you can use the command line utilities `curl` and `shasum` to compute the hash of a remote file:
  ```
  $ curl -L http://example.org/file.tar.gz | shasum -a 256
  ```
  replacing `http://example.org/file.tar.gz` with the actual URL of the file you want to download.

* For a [`GitSource`](@ref), the hash is the 40-character SHA1 hash of the revision you want to checkout.  For reproducibility you must indicate a specific revision, and not a branch or tag name, which are moving targets.

### Now that I have a published and versioned `jll` package, what compat bounds do I put in its dependents? What if the upstream does not follow SemVer?

Imagine there is a package `CoolFortranLibrary_jll` that is a build of an upstream Fortran library `CoolFortranLibrary`. We will abbreviate these to `CFL_jll` and `CFL`.

Once you have `CFL_jll` you might want to have a Julia project that depends on it.
As usual you put a compat bound for `CFL_jll` (the version number of upstream `CFL` and the jll version of `CFL_jll` are typically set equal during the `jll` registration).
If you know for a fact that upstream `CFL` follows SemVer, then you just set compat bounds as if it was any other Julia project.
However, not all ecosystems follow SemVer. The following two cases are quite common:

1. `CFL` releases version 1.1.1 and version 1.1.2 that are incompatible. A real world example is Boost (which breaks the ABI in every single release because they embed the full version number in the soname of libraries). If you have a typical permissive semver-style compat section in a package that depends on `CFL_jll`, then your package will break whenever `CFL_jll` gets a new release. To solve this issue you have to use "hyphen style" compat bounds like `"0.9.0 - 1.1.2"`. This leads to a separate problem: you need to change the compat bound every time there is a new `CFL_jll` release: this is the least bad option though -- it causes more annoyance for developers but it ensures users never end up with broken installs. And bots like `CompatHelper` can mostly automate that issue.
2. `CFL` releases versions 1.0.0 and 2.0.0 that are perfectly compatible. The Linux kernel, Chrome, Firefox, and curl are such examples. This causes annoying churn, as the developer still needs to update compat bounds in packages that depend on `CFL_jll`. Or, if you have a very strong belief in `CFL`'s commitment to backward compatibility, you can put an extremely generous compat bound like `">= 1.0.0"`.

While the SemVer (and Julia's) "conservative" approach to compatibility ensures there will never be runtime crashes due to installed incompatible libraries, you might still end up with systems that refuse to install in the first place (which the Julia ecosystem considers the lesser evil). E.g., package `A.jl` that depends on newer versions of `CLF` and package `B.jl` that depends on older versions can not be installed at the same time. This happens less often in ecosystems that follow semver, but might happen relatively often in an ecosystem that does not. Thus developers that rely on `jll` packages that do not follow semver should be proactive in updating their compat bounds (and are strongly encouraged to heavily use the `CompatHelper` bot).
