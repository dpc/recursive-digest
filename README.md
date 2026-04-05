# Recursive file-system digest

This library implements a simple but efficient recursive file-system digest
algorithm. You have a directory with some content in it, and you'd like
a cryptographical digest (hash) of its content.

It was created for the purpose of checksuming source code packages
in `crev`, but it is generic and can be used for any other purpose.

## Algorithm

Given any digest algorithm `H` (a Hash function algorithm),
a `RecursiveDigest(H, path)` is:

* for a file: `H("F" || file_content)`
* for a symlink: `H("L" || symlink_content)`
* for a directory: `H("D" || directory_content)`

As you can see a one-letter ASCII prefix is used to make it impossible
to create a file that has the same digest as a directory,
etc. The drawback of this approach is that `RecursiveDigest(H, path)` of
a simple file is not the same as just a normal digest of it (`H(file_content)`) .

`file_content` is just the byte content of a file.

`symlink_content` is just the path the symlink is pointing to, as bytes.

`directory_content` is created by:

* sorting all entries of a directory by name, in ascending order,
  using a simple byte-sequence comparison
* for all entries concatenating pairs of:
    * `H(entry_name)`
    * `RecursiveDigest(H, entry_path)`

If optional additional data extensions is used, the `H(entry_name)` above becomes
`H(entry_name || 0 || additional data)`. The format and meaning of additional
data is unspecified, but was intendet for fielsystem metadata like file system
permissions and ownership.

## Portability of non-UTF-8 paths

Filenames and symlink targets that are valid UTF-8 are hashed as their
UTF-8 bytes on every platform, so the resulting digest is bit-identical
across operating systems. This is the portable case and covers the
overwhelming majority of real-world trees.

Filenames and symlink targets that are **not** valid UTF-8 cannot be
hashed portably, because unix and Windows store them in fundamentally
different byte representations. In that case the library falls back to
the platform's native encoding:

* on unix, the raw bytes of the `OsStr` (`OsStrExt::as_bytes`);
* on Windows, the UTF-16 code units from `OsStrExt::encode_wide`
  encoded as little-endian bytes (this captures unpaired surrogates);
* on any other platform, the digest computation fails with
  `DigestError::OsStrConversionError`, because there is no defined
  byte representation to fall back to.

If you rely on cross-platform reproducibility of digests, treat any
tree containing non-UTF-8 path components as not portably
content-addressable.


