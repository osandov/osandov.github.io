---
layout: post
title: Making debuginfod Viable for the Linux Kernel
tags: debugging
---

[debuginfod](https://sourceware.org/elfutils/Debuginfod.html) is a service providing debugging symbols, source code, and executables via an HTTP API. Most Linux distributions run a debuginfod server, and many debugging tools make use of it automatically. debuginfod removes the need to manually install debugging symbols, which is a massive usability improvement for debuggers.

However, when I attempted to use debuginfod in [drgn](https://github.com/osandov/drgn), a programmable debugger primarily targeted at the Linux kernel, I ran into some server-side performance issues. Specifically, getting debugging symbols for the kernel and loaded kernel modules took *over an hour*, and most of that time was spent waiting for the debuginfod server to respond to queries.

This turned out to be caused by design decisions in Linux package management, compression algorithms, and debuginfod that are completely reasonable in isolation but interacted pessimally. By learning about these components, I was able to come up with a solution that reduces the time to get the same debugging symbols to *2 minutes*.

## How debuginfod Works

(Almost) every binary on Linux has a "build ID": a byte string that uniquely identifies it. Here's the build ID for a binary on my machine:

```console
$ readelf --notes /usr/bin/cat | grep "Build ID"
    Build ID: 4880bb013184e34a6a3ee3187e1d6282b6abcf2e
```

debuginfod queries are based on this build ID. For example, to get the debugging symbols for the above binary:

```console
$ curl https://debuginfod.fedoraproject.org/buildid/4880bb013184e34a6a3ee3187e1d6282b6abcf2e/debuginfo > cat.debug
```

On the server side, debuginfod periodically scans a set of directories looking for binaries and creates an index mapping build IDs to files. When it receives a query, it looks up the file with the given build ID and responds with its contents.

However, for Linux distributions, it's not practical to have a copy of every binary lying around for debuginfod. What distros *do* already have is a copy of every *package* (e.g., RPM or deb) containing those binaries. So, when debuginfod scans for binaries, it also checks for packages. It temporarily extracts each package it finds in order to create another index mapping build IDs to packages. To respond to a query, debuginfod looks up the package containing the given build ID and extracts the desired file.

## Linux Package File Formats

Software packages on Linux are generally glorified archive files with some extra metadata.

In particular, [RPM](https://rpm-software-management.github.io/rpm/manual/format_v4.html) files (used by Fedora, Red Hat Enterprise Linux, SUSE, and others) consist of a few metadata sections plus a compressed [cpio](https://en.wikipedia.org/wiki/Cpio) archive. The cpio archive contains the binaries and other files along with their metadata (name, owner, permissions, etc.). The cpio archive looks something like this internally:

<table>
    <thead>
        <tr>
            <th colspan="3">File 1</th>
            <th colspan="3">File 2</th>
            <th>File 3</th>
            <th>...</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Metadata</td>
            <td>Name</td>
            <td>Data</td>
            <td>Metadata</td>
            <td>Name</td>
            <td>Data</td>
            <td>...</td>
            <td>...</td>
        </tr>
    </tbody>
</table>

I.e., it is a flat list of file metadata, name, and data. Note that there's no index of files. Furthermore, this whole thing is compressed as a ["solid archive"](https://en.wikipedia.org/wiki/Solid_compression). As a result, to get a specific file, you have to check every entry until you find it, which also means decompressing everything up to that point.

[deb](https://en.wikipedia.org/wiki/Deb_(file_format)) files (used by Debian, Ubuntu, and others) are similar, although they use different formats: a deb file is actually an [ar](https://en.wikipedia.org/wiki/Ar_(Unix)) archive that contains, among other things, a compressed [tar](https://en.wikipedia.org/wiki/Tar_(computing)) archive. That compressed tar archive contains the files and their metadata. The exact cpio, ar, and tar formats are different, but they have the same overall structure.

## The Problem With Linux Kernel Packages

As noted earlier, debuginfod responds to queries by extracting the desired file from a package, which requires decompressing and checking every archive entry until the desired entry is found. Most packages only contain a handful of files, so this isn't too expensive. For example, the git-core-debuginfo package on Fedora is about 15 MB and contains 9 files. In contrast, the kernel-debuginfo package is almost 1 GB and contains over 4000 files, including the main kernel image (vmlinux) and numerous loadable kernel modules. Extracting a file from the end of the kernel-debuginfo package can therefore take a long time: over a minute in my tests.

debuginfod does have a couple of optimizations to mitigate this: it caches recently used files so that they don't need to be extracted again, and it prefetches a few extra entries from the same package in case they're needed, too. However, systems usually have tens of kernel modules loaded, which vary greatly by hardware and workload. If even a few of them are uncached and toward the end of the archive, user experience suffers greatly.

## Random-Access Reading in xz

If debuginfod could skip to the desired file in an archive instead of needing to decompress everything before it, then this would be much faster. Most compression formats (e.g., gzip, bzip2, and Zstandard) don't support this: in order to decompress a specific byte, you normally need to decompress every byte before it first. But, there is one widely-used compression format that supports random access: [xz](https://en.wikipedia.org/wiki/XZ_Utils). And, even better, the kernel-debuginfo package in the Fedora/RHEL ecosystem and the linux-image-dbg package in the Debian/Ubuntu world are already compressed with xz!

Surprisingly, liblzma, the reference implementation of xz, doesn't have a convenient API for random-access reading. To use it, we actually need to understand the [.xz file format](https://tukaani.org/xz/format.html).

Random access in xz works by splitting the input stream into multiple, independently compressed blocks:

<table>
    <thead>
        <tr>
            <th colspan="6">Stream</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Stream Header</td>
            <td>Block 1</td>
            <td>Block 2</td>
            <td>...</td>
            <td>Index</td>
            <td>Stream Footer</td>
        </tr>
    </tbody>
</table>

The compressed xz stream ends with an index of block records:

<table>
    <thead>
        <tr>
            <th colspan="7">Index</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Index Indicator</td>
            <td>Number of Records</td>
            <td>Record 1</td>
            <td>Record 2</td>
            <td>...</td>
            <td>Index Padding</td>
            <td>CRC32</td>
        </tr>
    </tbody>
</table>

Each record stores the compressed and uncompressed sizes of one block:

<table>
    <thead>
        <tr>
            <th colspan="2">Record</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Compressed Size</td>
            <td>Uncompressed Size</td>
        </tr>
    </tbody>
</table>

The offset of a block in the compressed stream is the sum of the compressed size of every block before it, and likewise for the uncompressed offset. To read starting from a specific uncompressed offset, you must:

1. Use the index to find the block containing the uncompressed offset.
2. Seek to the block's compressed offset in the file.
3. Calculate the difference between the target offset and the block's uncompressed offset.
4. Decompress and throw away that many bytes.
5. Decompress and return the remaining bytes.

Note that this isn't true random access: you still have to decompress some unneeded data. But, if the block size is reasonable, you can avoid a lot of unnecessary decompression. On a recent Fedora kernel-debuginfo RPM, the uncompressed block size, and thus the worst case amount of unnecessary decompression, is 12 MB, which is a couple of orders of magnitude smaller than the entire 1 GB file.

It's also important to note that xz compression defaults to one block, i.e., no random access. However, multi-threaded compression necessarily splits the input into blocks to compress independently, so anything compressed with multi-threaded xz gets random access for free.

I developed a [patch series](https://lore.kernel.org/linux-debuggers/cover.1721773977.git.osandov@fb.com/) for debuginfod to make use of random-access reading from xz archives when possible. It will be available in elfutils 0.192.

## Future Work

The biggest downside of this approach is that it depends on a packaging implementation detail. The Fedora and Debian developers chose multi-threaded xz for its performance, not because they wanted a format that supports random access. At the very least, I plan to contact the package maintainers and document this so that debuginfod is considered before changing compression types.

Specifically, Zstandard compression, which does not support random access, is now becoming the norm. There is an [experimental seekable format](https://github.com/facebook/zstd/blob/dev/contrib/seekable_format/zstd_seekable_compression_format.md), but it is implemented in a separate library and is not guaranteed to be stable. It is [not currently a priority](https://github.com/facebook/zstd/issues/395#issuecomment-2048888278) for the Zstandard developers because they haven't found a compelling enough use case. Perhaps the debuginfod use case would qualify, but I haven't done any measurements of how much of an improvement it would be, and the xz status quo is acceptable.

There is also the possibility of using an indexed, non-solid archive format in package files. [ZIP files](https://en.wikipedia.org/wiki/ZIP_(file_format)), for example, compress each file separately and have a "central directory" listing all files. However, formats like this compress less efficiently, and a lot of tooling expects the current archive formats. A good middle ground might be to keep the current formats but align xz block boundaries to files or groups of files over a certain size threshold.

## Conclusion

This was an interesting problem slightly outside of my usual domain, so it was very satisfying to come up with a solution. The debugging experience keeps getting better thanks to new tools and infrastructure like debuginfod, but there's still a lot of exciting work to do.
