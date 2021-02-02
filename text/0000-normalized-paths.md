- Feature Name: normalized_paths
- Start Date: 2020-10-28
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

The current path module lacks methods for lexical path normalization. Compared
to canonicalization, lexical path processing:

- Does not rely on system calls to return a normalized path.
- Allows to process path to files or directories that do not exist on the
  filesystem.
- Removes the need to handle I/O errors.

These features are the building block of some other common operations, such as
finding the relative path between two paths, or ensuring a path is restricted
to some base directory.

# Motivation
[motivation]: #motivation

The main goal is to implement most of the missing features compared to the
paths modules in Python and Go. The following subsections outline some
limitations of Rust path module.

## What if I have a folder named `C:`?

Trying to get `C:\Users\user\Documents\C:\foo` using `Path::join` yields an
absolute path:

```rust
assert_eq!(
    Path::new("C:\Users\user\Documents").join("C:\foo"),
	Path::new("C:\foo"),
);
```

Although paths are represented by strings, `Path::join` is a high-level method
that processes its argument to determine if it is absolute. The fundamental
operation of appending a path to another by string concatenation is called a
lexical join.

`Path::join` is a refinement of lexical joins:

```rust
impl Path {
	fn join<P: AsRef<Path>>(&self, path: P) -> PathBuf {
		let path = path.as_ref();

		if path.is_absolute() {
			return path.into();
		}

		return self.lexical_join(path);
	}
}
```

## What if I want to prevent path traversal?

Web servers are exposed to path traversal vulnerabilities that allow an
attacker to access file outside of some base directory. `Path::join` from the
base directory `/srv` with a user-supplied path can yield a path outside of
`/srv`:

```rust
assert_eq!(
	Path::new("/srv").join("/etc/passwd"),
	Path::new("/etc/passwd")
);
```

Only accepting relative paths is not enough:

```rust
assert_eq!(
	Path::new("/srv").join("../etc/passwd"),
	Path::new("/etc/passwd")
);
```

Stripping `..` prefixes is not enough either:

```rust
assert_eq!(
	Path::new("/srv").join("foo/../../etc/passwd"),
	Path::new("/etc/passwd")
);
```

If the user-provided location only needs to be a single path component, the
programmer can forbid any string containing paths separators. Otherwise, the
inner `..` needs to be collapsed with their parent directory, which is a
feature of normalization.

## What if I canonicalize all the things?

Let's say the user-supplied path from the previous web server is used to create
a new file instead:

```rust
Path::new("/srv")
	.join("new_file.txt")  // -> /srv/new_file.txt
	.canonicalize()?       // -> Err("No such file or directory")
	.starts_with("/srv/");
```

The error "No such file or directory" is returned by `Path::canonicalize`
because it needs a concrete path (a path that refers to an existing file or
directory on the file system).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The following subsections introduce various lexical path processing methods.

## Definitions

There are two ways to join paths:

- Regular: if the second path is absolute, it replaces the first, otherwise
  they are lexically joined together. This is the behavior of `Path::join`.
- Lexical: string-like concatenation, with a path component separator in
  between.

There are three ways to transform a path into a shorter equivalent:

- Normalization: lexically eliminate duplicate `/`, `.`, and `..` components.
  The path is in the normal form.
- Canonicalization: fetch the canonical path from system APIs (`realpath` on
  Unix requires a concrete file or directory, and `GetFullPathName` on Windows
  without this requirement *TODO: is it true?*). The path is in the canonical
  form, which implies the normal form.
- Hybrid: canonicalize each component while possible, then use normalization
  for the remaining ones. The path is in the proximate form, which implies
  the normal form.

A canonical path is normalized, but a normalized path is not necessarily
canonical (for example, it can still be relative).

## Absolute

`Path::absolute` returns the absolute equivalent of `self`:

```rust
impl Path {
    fn absolute<P: AsRef<Path>>(&self) -> Result<PathBuf>;
}
```

If `path` is relative, the absolute equivalent is obtained by joining the
current working directory (CWD), obtained from `std::env::current_dir`, with
`path`.

### Examples

If the CWD is `/home/user`:

```rust
assert_eq!(
    Path::new("/usr").absolute(),
	Ok(Path::new("/usr"))
);

assert_eq!(
    Path::new(".").absolute(),
	Ok(Path::new("/home/user/."))
);

assert_eq!(
    Path::new("..").absolute(),
	Ok(Path::new("/home/user/.."))
);
```

## Is inside

`rooted_join` constructs a path restricted to some base, whereas `is_inside`
tells if a path is a descendant of `base`:

`Path::is_inside` returns whether `self` is a lexical descendant of `base`:

```rust
impl Path {
    fn is_inside<P: AsRef<Path>>(&self, base: P) -> bool;
}
```

Before comparison, both `self` and `path` are normalized. If this method
returns true, `path` is guaranteed to be lexically inside `self`.

### Limitations

The same limitations as `Path::normalize` apply. To ensure that `self` is
concretely inside `base` (on the filesystem), canonicalize `self` before
calling `is_inside`.

### Examples

TODO: Add examples

## Normalization

`Path::normalize` returns the normalized equivalent of `self`:

```rust
impl Path {
    fn normalize<P: AsRef<Path>>(&self) -> PathBuf;
}
```

The returned path is the shortest equivalent, normalized by pure lexical
processing using the following rules:

1. Replace duplicate `PATH_SEP` with single ones.
2. Eliminate `.` path components (referring to the current directory).
3. Collapse inner `..` path components (referring to the parent directory) with
   a previous component or against the root.

### Differences with `Path::canonicalize`

This method is the lexical counterpart of `Path::canonicalize`, but it does not
interact with the filesystem:

1. It does not resolve symlinks.
2. It does not require existing files or directories.
3. It returns a relative path when a relative path is given.

### Limitations

Canonicalization resolves path components against the filesystem, yielding the
true absolute (canonical) path, whereas lexical normalization can change the
meaning of a path and point to another file or directory under some specific
circumstances.

If `/a/b` is a symlink to `/d/e`, given `/a/b/../c`:

- [`Path::canonicalize`] returns `/d/c`.
- [`NormPathExt::normalize`] returns `/a/c`.

### Examples

```rust
assert_eq!(
    Path::new("/usr/.//lib/../../var"),
    Path::new("/var")
);

assert_eq!(
    Path::new(".././foo/bar/.."),
    Path::new("../foo")
);
```

## Lexical join

`Path::lexical_join` returns `self` lexically joined to `path`:

```rust
impl Path {
    fn lexical_join<P: AsRef<Path>>(&self, path: P) -> PathBuf;
}
```

`PathBuf::lexical_push` lexically pushes `path` onto `self`:

```rust
impl PathBuf {
    fn lexical_push<P: AsRef<Path>>(&mut self, path: P);
}
```

### Differences with `Path::join` and `PathBuf::push`

These methods are the lexical counterparts of `Path::join` and `PathBuf::push`.
They join `self` to `path` in the sense of string concatenation, adding the
native path separator in between only if necessary. If the given `path` is
absolute, it does not replace `self`.

### Examples

```rust
assert_eq!(
    Path::new("/home/user").lexical_join("downloads"),
    Path::new("/home/user/downloads")
);

assert_eq!(
    Path::new("/home/user").lexical_join("/downloads"),
	Path::new("/home/user/downloads")
);

assert_eq!(
    Path::new("C:\Users\User").lexical_join("C:\Documents"),
	Path::new("C:\Users\User\C:\Documents")
);
```

## Relative

`Path::relative_to` returns the relative path from `base` to `self`:

```rust
impl Path {
    fn relative_to<P: AsRef<Path>>(&self, base: P) -> Option<PathBuf>;
}
```

It returns `None` if determining the relative path by pure lexical processing
is impossible, such as when the CWD is needed.

### Examples

```rust
assert_eq!(
    Path::new("/usr/lib").relative_to("/usr"),
	Some(PathBuf::from("lib"))
);

assert_eq!(
	Path::new("usr/bin").relative_to("var"),
	Some(PathBuf::from("../usr/bin"))
);

assert_eq!(
    Path::new("foo").relative_to("/"),
	None
);

assert_eq!(
    Path::new("foo").relative_to(".."),
	None
);
```

## Rooted join

`Path::rooted_join` returns `path` rooted at `self`:

```
impl Path {
    fn rooted_join<P: AsRef<Path>>(&self, path: P) -> PathBuf;
}
```

It ensures that `path` is restricted to `self`, preventing path traversal
outside of `self`. The returned path `rel` verifies: `base.join(rel) == self`.

### Limitations

The same limitations as `Path::normalize` apply in the presence of symlinks:
the returned path may refer to a symlink that points to a file or directory
outside of `self`.

Additionally, this method does not allow to exit and then to re-enter the base
directory, because it could be used to guess the parent path components, and it
does not work for relative paths (the CWD is needed to re-enter):

```rust
assert_eq!(
    Path::new("/srv").rooted_join("../srv/file.txt"),
    Path::new("/srv/srv/file.txt")
);
```

### Examples

```rust
assert_eq!(
    Path::new("/srv").rooted_join("file.txt"),
    Path::new("/srv/file.txt")
);

assert_eq!(
    Path::new("/srv").rooted_join("/file.txt"),
    Path::new("/srv/file.txt")
);

assert_eq!(
    Path::new("/srv").rooted_join("../file.txt"),
    Path::new("/srv/file.txt")
);

assert_eq!(
    Path::new("/srv").rooted_join("foo/../file.txt"),
    Path::new("/srv/file.txt")
);

assert_eq!(
    Path::new("/srv").rooted_join("foo/../../file.txt"),
    Path::new("/srv/file.txt")
);
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The purpose of this section is to give some insights about some issues that
arise with path handling and which features are implemented by existing
implementations or platforms.

## Windows

Windows has some specificities in the way it handles paths. This sections
highlights some elements from [File path formats on Windows
systems](https://docs.microsoft.com/en-us/dotnet/standard/io/file-path-formats).

### Paths types

> A standard DOS path can consist of three components:
>
> - A volume or drive letter followed by the volume separator (`:`).
> - A directory name. The directory separator character separates
>   subdirectories within the nested directory hierarchy.
> - An optional filename. The directory separator character separates the file
>   path and the filename.

Windows maintains per-volume current directories. `env::current_dir` only
fetches the current directory on the current drive, which is a limitation to
`NormPathExt::absolute` implementation.

> For example, if the file path is `D:sources`, the current directory is
> `C:\Documents\`, and the last current directory on drive `D:` was
> `D:\sources\`, the result is `D:\sources\sources`.

Some paths are not valid, for instance, non-fully qualified UNC paths, or the
empty path (`GetFullPathName` returns an error). Rust does not match this
behavior (TODO: check this).

> UNC paths must always be fully qualified. They can include relative directory
> segments (. and ..), but these must be part of a fully qualified path. You
> can use relative paths only by mapping a UNC path to a drive letter.

> DOS device paths are fully qualified by definition. Relative directory
> segments (. and ..) are not allowed. Current directories never enter into
> their usage.

### Normalization

> A path that begins with a legacy device name is always interpreted as a
> legacy device by the `Path.GetFullPath(String)` method. For example, the DOS
> device path for `CON.TXT` is `\\.\CON`, and the DOS device path for
> `COM1.TXT\file1.txt` is `\\.\COM1`.

> All forward slashes (`/`) are converted into the standard Windows separator,
> the back slash (`\`). If they are present, a series of slashes that follow
> the first two slashes are collapsed into a single slash.

All forward slashes (`/`) are converted to backslashes (`\`). Rust stdlib only
parses Windows path prefixes with backslashes (check if it is not done in
`Path::new`).

Some characters are trimmed:

> - If a segment ends in a single period, that period is removed. (A segment of
>   a single or double period is normalized in the previous step. A segment of
>   three or more periods is not normalized and is actually a valid
>   file/directory name.)
> - If the path doesn't end in a separator, all trailing periods and spaces
>   (U+0020) are removed. If the last segment is simply a single or double
>   period, it falls under the relative components rule above.

Skip normalization for paths starting with `\\?\` passed to Windows APIs (TODO:
consequences on the Rust API that must use normalization?):

> - To get access to paths that are normally unavailable but are legal. A file
>   or directory called hidden., for example, is impossible to access in any
>   other way.
> - To improve performance by skipping normalization if you've already
>   normalized.

TODO: [GetFullPathName](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-getfullpathnamea)

### Letter case

> Directory and file name comparisons are case-insensitive. If you search for a
> file named `test.txt`, .NET file system APIs ignore case in the comparison.
> `Test.txt`, `TEST.TXT`, `test.TXT`, and any other combination of uppercase
> and lowercase letters will match `test.txt`.

Comparing `Component` is case-sensitive, whereas Windows paths are
case-insensitive:

## Normpath

The implementation tries to avoid normalization unless necessary. For instance,
where Go's `filepath.Join` returns a normalized path,
`NormPathExt::lexical_join` only adds a separator when needed.

It is theoretically possible to implement normalization without extra space
when the original buffer is large enough.

The implementation deviates from platform specific behavior.

### Absolute

- Take current working directory from env (it appears the variables are =C,
  =D).

# Drawbacks
[drawbacks]: #drawbacks

1. Implementing everything under a single `Path` type can add confusion by
   adding more ways of doing similar operations, but with subtle differences.
2. Although the lexical processing does not depend on the platform, having
   everything under the same `Path` type prevents the manipulation of Windows
   paths on Unix (no access to the prefixes).
3. Lexical processing is independent from the filesystem, and can often alter
   the meaning of a path with ".." components and symlinks. It can lead to
   inconsistency issue.
4. Some operations such as normalization can be performed in-place (such as
   when the path is already normalized). The Rust implementation always
   allocates and returns a new buffer.
5. There might be better designs, implementations, or alternatives (see the
   next section).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The approach outlined in this RFC was chosen for the following reasons:

1. It is inspired by proven standard libraries from other languages.
2. Normalization is done only when required.
3. It adds a restricted amount of methods required to make normalization work
   with some core features.
4. The new methods can be used seamlessly with the existing ones.
5. The "toolbox" approach allows users to mix and match different primitives to
   make their own functions. (TODO: example, such as passing canonicalized
   paths to is inside).
6. rooted\_join implements an operation that improve security in paths
   handling, in line with Rust safety philosophy.
7. The existing semantics of the `Path` type are unchanged.
8. The design is cross-platform.
9. It can be extended with the implementation of other features that rely on
   lexical processing. (See: [Future possibilities](#future-possibilities)).
10. It matches the low-level path module in Python.

An alternative implementation could use different Path types to represent pure
paths on each platform, akin to Python high-level `pathlib` module. A Rust
implementation could use a `PurePathExt` trait for cross-platform pure
operations as an alternative to Python's inheritance (the implementation is
left as an exercise for the reader).

Advantages of this alternative design:

- There is a clear distinction between pure paths (with lexical operations),
  and concrete paths on the filesystem (with system calls).
- `PureUnixPath` and `PureWindowsPath` can both be used on either Unix or
  Windows.
- Users can just use the `Path` type and forget about the others.

Drawbacks:

- It is more complex.
- There is a lot more stuff (12 path types + 3 trait?).

# Prior art
[prior-art]: #prior-art

In [Lexical File Names in Plan 9](https://9p.io/sys/doc/lexnames.html), Rob
Pike outlines Unix paths issues with symlinks. The paths are non-hierarchical,
and pure lexical processing does not allow to perform certain operations. The
article explains Plan 9 design decisions and its definition of the parent
directory.

## Rust

The issue has already been raised in the Rust project, in different forms:

- [Equivalent to Python's
  os.path.normpath](https://github.com/rust-lang/rfcs/issues/2208)
- [Cross-platform file system
  abstractions](https://github.com/rust-cli/meta/issues/10)
- [\[Pre-RFC\] Additional path handling
  utilities](https://internals.rust-lang.org/t/pre-rfc-additional-path-handling-utilities/8405)
- [Add function to make paths absolute, which is different from
  canonicalization](https://github.com/rust-lang/rust/issues/59117): this issue
  contains a number of links to possibly linked issues or discussions in
  various Rust projects.

Some Rust libraries that implement similar features:

- [relative\_path](https://docs.rs/relative-path/1.3.2/relative_path):
  implement pure path operations, including `Path::normalize` and
  `Path::relative_to`, for portability purpose.
- [path\_abs](https://docs.rs/path_abs/0.5.0/path_abs/): non-canonicalized pure
  absolute paths (they may refer to non-existing file or directories). Most
  notably, there is an implementation of `Path::lexical_join` and
  `Path::normalize`.
- [libpathrs](https://docs.rs/pathrs/0.0.2/pathrs/): wrappers around the
  `openat2` system call. The idea is similar to `Path::rooted_join`, except it
  relies on system API to ensure the path is relative to some root.
- [lexical\_paths](https://github.com/gdzx/lexical_paths): implements this RFC.

In the wild:

- [Cargo
  `normalize_path`](https://github.com/rust-lang/cargo/blob/master/src/cargo/util/paths.rs#L61):
  partial implementation of `Path::normalize`.
- [Rocket URI
  segments](https://github.com/SergioBenitez/Rocket/blob/master/core/http/src/uri/segments.rs#L65):
  path "normalization" in the context of URIs.
- [Overly strict FromSegments implementation for
  PathBuf](https://github.com/SergioBenitez/Rocket/issues/560#issuecomment-364173013):
  the previous normalization method is implemented in a simple but overly
  strict way.

## Other languages

Several languages provide similar features in their standard library:

|              | C++                 | C#/.NET         | Go           | Java           | Node.js   | Python   | Ruby                          | Rust         | Rust + lexical\_paths         |
|--------------|---------------------|-----------------|--------------|----------------|-----------|----------|-------------------------------|--------------|-------------------------------|
| Absolute     | absolute            |                 | Abs          | toAbsolutePath |           | abspath  | File.absolute\_path (Core)    |              | absolute                      |
| Canonicalize | canonical           | GetFullPath     | EvalSymlinks | toRealPath     | resolve   | realpath | Pathname.realpath             | canonicalize | canonicalize                  |
| Relative     | lexically\_relative | GetRelativePath | Rel          | relativize     | relative  | relpath  | Pathname.relative\_path\_from |              | relative\_to                  |
| Join         | append              | Combine         |              | resolve        | join      | join     | Pathname.join                 | push / join  | push / join                   |
| Lexical join | concat              | Join            | Join         | of             |           |          | File.join (Core)              |              | lexical\_push / lexical\_join |
| Normalize    | lexically\_normal   |                 | Clean        | normalize      | normalize | normpath | Pathname.cleanpath            |              | normalize                     |
| Resolve      | weakly\_canonical   |                 |              |                |           | ~abspath |                               |              |                               |

The following subsections gives their implementation highlight.

### C++

#### libstdc++

- Docs
  - [`filesystem`](https://en.cppreference.com/w/cpp/filesystem)
  - [`filesystem::path`](https://en.cppreference.com/w/cpp/filesystem/path)
- [Sources](https://github.com/gcc-mirror/gcc/blob/master/libstdc++-v3/src/c++17/fs_ops.cc)

#### Highlights

[C++ Relative Paths for
Filesystem](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0219r1.html)
is a design document for this module.

`path` defines the following lexical methods:

- [`path::lexically_normal()`](https://en.cppreference.com/w/cpp/filesystem/path/lexically_normal):
  returns the normalized equivalent of `this` (normal form).
- [`path::lexically_relative(base)`](https://en.cppreference.com/w/cpp/filesystem/path/lexically_normal):
  returns `this` relative to `base` (relative form).
- [`path::lexically_proximate(base)`](https://en.cppreference.com/w/cpp/filesystem/path/lexically_normal):
  returns `this` relative to `base`, or `this` if impossible (proximate form).

The counterparts of `path.lexically_normal()` in the `filesystem` module are:

- [`filesystem::canonical(path)`](https://en.cppreference.com/w/cpp/filesystem/canonical):
  return the true absolute (canonical) path. The entry must exist on the
  filesystem and symlinks are followed. The result is in normal form.
- [`filesystem::weakly_canonical(path)`](https://en.cppreference.com/w/cpp/filesystem/canonical):
  returns the canonical path for leading elements that exist, joined with the
  normal equivalent of the remaining components.

The counterparts of `path::lexically_relative(base)` and
`path::lexically_proximate(base)` in the `filesystem` module are:

- [`filesystem::relative(path,
  base)`](https://en.cppreference.com/w/cpp/filesystem/relative): returns
  `weakly_canonical(path).lexically_relative(weakly_canonical(base))`.
- [`filesystem::proximate(path,
  base)`](https://en.cppreference.com/w/cpp/filesystem/relative): returns
  `weakly_canonical(path).lexically_proximate(weakly_canonical(base))`.

When `base` is not provided, it is substituted by the CWD.

Differences in the normalization algorithm:

- The normalization preserves trailing `/.` (TODO: example).

### C#/.NET

#### `System.IO`

- [Docs](https://docs.microsoft.com/en-us/dotnet/api/system.io.path?view=net-5.0)
- [Sources](https://github.com/dotnet/runtime/tree/master/src/libraries/System.Private.CoreLib/src/System/IO)
  - `Path.cs` for the public API
  - `Path.Unix` / `Path.Windows`
  - `PathInternal`

#### Highlights

- Distinguishes from fully qualified path and rooted path.

### Go

#### `path/filepath`

- [Docs](https://golang.org/pkg/path/filepath/)
- [Sources](https://golang.org/src/path/filepath/)

#### Highlights

- Cross-platform module.
- Paths are processed like strings.
- Relies heavily on normalization, that is most methods return normalized path
  and use normalization internally. The process is optimized to avoid
  allocation unless necessary with a lazy buffer that is cloned on mismatch.
- No join like `Path::join`.

### Java

#### `java.base.Path`

- [Docs](https://docs.oracle.com/en/java/javase/15/docs/api/java.base/java/nio/file/Path.html)
- Sources:
  - [`UnixFileSystem.java`](https://github.com/openjdk/jdk/blob/master/src/java.base/unix/classes/java/io/UnixFileSystem.java)
  - [`WinNTFileSystem.java`](https://github.com/openjdk/jdk/blob/master/src/java.base/windows/classes/java/io/WinNTFileSystem.java)

#### Highlights

- Cross-platform module.
- Confusing usage.
- Paths come from a filesystem implementation (no pure concept).

### Node.js

#### `path`

- [Docs](https://nodejs.org/api/path.html)
- [Sources](https://github.com/nodejs/node/blob/master/lib/path.js)

#### Highlights

- Cross-platform module:
  - `path.posix`: POSIX paths.
  - `path.win32`: Windows paths.
- `path.resolve(paths..)`: resolves a sequence of paths or path segments into
  an absolute path (using `CWD` and squashing absolute paths). It's an all-in
  one absolute, current\_dir, and canonicalize.
- `path.relative(from, to)`: returns the relative path based on the current
  working directory.

### Python

The main difference with the `os.path` module is the handling of `//` prefixes.
From the [POSIX specification](https://pubs.opengroup.org/onlinepubs/000095399/basedefs/xbd_chap04.html#tag_04_11):

> A pathname that begins with two successive slashes may be interpreted in an
> implementation-defined manner, although more than two leading slashes shall
> be treated as a single slash.

- [Django `safe_join`](https://github.com/django/django/blob/master/django/utils/_os.py)

#### Low-level `os.path`

- [Docs](https://docs.python.org/3/library/os.path.html)
- Sources:
  - [`posixpath`](https://github.com/python/cpython/blob/master/Lib/posixpath.py)
  - [`ntpath`](https://github.com/python/cpython/blob/master/Lib/ntpath.py)

##### Highlights

- Cross-platform modules

#### High-level `pathlib`

- [Docs](https://docs.python.org/3/library/pathlib.html)
- [Sources]()

![Python pathlib](https://docs.python.org/3/_images/pathlib-inheritance.png)

##### Highlights

- Pure paths: can be instantiated on any platform and no system calls are
  made.
- Concrete paths: depend on the local platform:
  - `Path` is either `PosixPath` or `WindowsPath`.
  - `PosixPath` and `WindowsPath` can only be instantiated on their respective
        platform.

The types are split by features:

- Pure path types implement only pure operations.
- Concrete path types implement all operations (including pure).

And by platform:

- Posix path types implement cross-platform operations and POSIX specific ones.
- Windows path types implement common operation and Windows specific ones.

All types except `PosixPath` and `WindowsPath` can be instantiated on any
platform (the reason is that they implement concrete platform-specific
operations). On the contrary, `PurePosixPath` can be instantiated on Windows
since all operations are lexical.

### Ruby

#### Core

- [Docs](https://ruby-doc.org/core-3.0.0/File.html)
- [Sources](https://github.com/ruby/ruby/blob/v3_0_0/file.c)

#### `pathname`

- [Docs](https://ruby-doc.com/stdlib/libdoc/pathname/rdoc/Pathname.html)
- [Sources](https://github.com/ruby/ruby/blob/master/ext/pathname/lib/pathname.rb)

#### Highlights

- Weird docs and implementations

### Rust

#### `std::path`

- [Docs](https://doc.rust-lang.org/std/path/index.html)
- Sources:
  - [`path.rs`](https://github.com/rust-lang/rust/blob/master/library/std/src/path.rs)
  - [`sys/unix/path.rs`](https://github.com/rust-lang/rust/blob/master/library/std/src/sys/unix/path.rs)
  - [`sys/windows/path.rs`](https://github.com/rust-lang/rust/blob/master/library/std/src/sys/windows/path.rs)

#### Highlights

- Cross-platform module
- `Path::components` iterator strips "." (has effects on basename, dirname)
- Windows path do not parse forward prefixes (maybe because if that appeared in
  Linux paths that would cause problems, especially for drive stuff). Would
  probably need a method to convert from/to forward slashes.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- The API around normalization could return `Cow<'a, Path>` to avoid copying
  paths.
- Issues with Windows paths?
- How to test it + edge cases (discrepancies between Python and Go
  normalization)
- Methods names

# Future possibilities
[future-possibilities]: #future-possibilities

The features can be mixed and matched in various way:

- Canonicalization and fallback to normalization (resolve).
- Try variants that return a sensible rooted path.
- Other tools to treat paths more like string (and easily join to find the
  original one), POSIX docs links.
- Common prefix + strip common prefix
