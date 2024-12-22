---
title: How to copy files from dependencies to your zig application
date: 2024-12-22T16:00:00+0900
description: Using namedWriteFiles
categories:
  - blog
tags:
  - zig
---
# Intro

One of the characteristics things of zig is its build system, which, coming from another language, might be confusing to newcomers. Here I'll introduce a part of zig that I found confusing, which is `namedWriteFiles`. 

In addition, I'll use this case to describe my learning process, to demonstrate how to work with zig despite the relative lack of guides or tutorials online. Zig has a very readable standard library and so it's easy to figure out things for yourself, making it very enriching (for the types who like that sort of thing).

# Motivation

If you've worked with zig for any amount of time, you probably have tried to add a third-party package and have come across `b.dependency`. I've seen many repos that explain how to add their files to your project, usually in some form like:

```md
# Install

// add to your build.zig.zon:
.dependencies = .{
    .some_dependency = .{
        .url = "..."
        .hash = "..."
    }
    ...
}

// build.zig
const some = b.dependency("some", .{...})
exe.root_module.addImport("some", some.module())
exe.linkLibrary(some.artifact("lib"))

```

And zig-only modules this usually works out alright.

The problem that I ran into is writing a zig module for bindings to SDL3. I wanted to use SDL as a pre-compiled, shared library that I packaged as a .dll and .lib in my project. My goal was **to expose this library through a dependency that I could reference from my main build.**

Other zig libraries that need to run complicated build logic usually have dependers do some form of this:

```zig
const std = @import("std");
const some = @import("some-dependency");

pub fn build(b: *std.Build) !void {
	const other = b.dependency("other-dependency");
	some.setup(b, .{...});
}

```

In my opinion (and feel free to disagree), I had some issues with this API.
- Passing in the root `build.zig` into a module, and giving it knowledge of the root `build.zig`, runs backward to a depender-dependee relationship. A library has no knowledge of the root build environment, so in order to run `setup()` I'd have to pass (in the worst case) my entire build configuration into a dependency. If I had some other configuration that conflicted with the build helper functions provided in the dependee, I'd have to fork the repo and maintain my own version.
- I wanted to keep things simple and interact with a dependency through the provided interface, `b.dependency()`.

# Write Files

So I trawled through the zig source code and saw `Dependency` exposed five functions:
- `artifact` - usually C/C++ libraries built from source.
- `module` - Zig code that can be `@import`ed.
- **`namedWriteFiles`**
- `namedLazyPath` - generated files that can be `@import`ed
- `path` - for [non-zig dependencies](https://github.com/ziglang/zig/commit/8eff0a0a669dbdacf9cebbc96fdf20536f3073ee)

The one most applicable to my use case seemed to be `namedWriteFiles` (having "file" in the name), but as a relative zig dumb-dumb I had no idea what it was even doing or how to use it. Maybe you, like I was, are also wondering exactly what the hell a "write file" is. There's not a lot of documentation on it because it's relatively newer, added October 2023 as a part of zig 0.13.0.

A `Step.WriteFile` in zig is a build step just like `Step.Compile` and `Step.InstallArtifact`: It creates a step in the build dependency graph that "writes" a "file" to the output directory. I've come to realize that specifically, "file" means to create a file of the following kinds:

| Function        | Usage       |
| --------------- | ----------- |
| `WriteFile.add`             | byte array  |
| `WriteFile.addCopyFile`      | files       | 
| `WriteFile.addCopyDirectory`  | directories |

In addition, a `WriteFile` can counter-intuitively contain multiple files and directories. So rather than every file using its own step, every file that needs to be added to the build graph is appended to one `Step.WriteFile`. And that leaves us with what "write" means.

The build system, simply, will cache outputs of generated files and only build dependencies that are stale. A `Step.WriteFile` will add a file to the zig cache only when the source is updated. Obviously this is useful in speeding up builds, but it does mean that zig builds don't write files  directly into the output directory - some indirection is needed.

# Usage

My `sdl` packages exposes (aka copies to cache) its pre-compiled dlls like this:

```zig
const dll_wf = b.addNamedWriteFiles("dlls");
_ = dll_wf.addCopyFile(b.path("lib/SDL3.dll"), "SDL3.dll");
_ = dll_wf.addCopyFile(b.path("lib/spirv-cross-c-shared.dll"), "spirv-cross-c-shared.dll");
if (dxc_enabled) {
	_ = dll_wf.addCopyFile(b.path("lib/dxil.dll"), "dxil.dll");
	_ = dll_wf.addCopyFile(b.path("lib/dxcompiler.dll"), "dxcompiler.dll");
}
```

And in my root package, I install (aka copy to build dir) the files from the cache:

```zig
b.installDirectory(.{
	.install_dir = .bin,
	.source_dir = sdl.namedWriteFiles("dlls").getDirectory(),
	.install_subdir = "",
});
```

# Thoughts

When integrating with C libraries, it seems to be preferred to compile from source and port the build system to zig. For some libraries, however, this is not trivial and so compiling a binary and adding a `WriteFile` step seems like a good way to integrate third-party code.