---
title: "exports Is a Deny List: Shipped in the Tarball, Unreachable to Import"
collection: journal
permalink: /journal/npm-exports-is-deny-by-default/
excerpt: "A named export from the SDK's package root resolved to undefined. Importing it directly from its own subpath threw a hard resolution error. The compiled file was sitting right there in the published tarball the whole time."
---

**The trick:** a package's `exports` map is a deny list — presence in the published tarball grants nothing. Install the built tarball fresh in a scratch project and import every path you claim to support.

## Issue

A named export from the SDK's package root resolved to `undefined` at runtime — no error, just a missing value where a class should have been. Trying to import the same thing from its own dedicated subpath instead produced a hard module-resolution error, naming the exact subpath as unreachable.

The compiled JavaScript for both was sitting in the published package the entire time.

## Root Cause

Two independent gates, each individually reasonable, stacked into total unreachability.

**The package's root barrel never re-exported the module.** A barrel file that forgets to re-export something doesn't error — it just resolves the name to nothing, because as far as the module system is concerned, no such export was ever declared.

**The package's export map had no entry for the subpath.** Modern package publishing lets you declare exactly which internal paths are importable from outside the package — everything not listed is unreachable, regardless of whether the file physically exists in the tarball. This map is a deny list by default: presence on disk grants nothing.

The package also straddles two module systems, shipping both an ESM build and a CommonJS build with separate type declarations for each — so a subpath entry needs matching conditions for both, and a half-declared one breaks one consumer type or the other silently rather than failing loud for everyone.

## Solution

Re-export the module from the root barrel, and add the missing subpath to the export map with both the ESM and CommonJS conditions and their respective type declarations.

The verification habit that stuck: package the tarball, install it fresh in a scratch project, and import all the paths that are supposed to work — root, subpath, and type-only — from the installed artifact, not from the source tree. What's in `dist/` is not what's shipped to users; what a clean install resolves is.

## 💡 Takeaway

- **An export map is an allowlist, not a description of what's in the package.** A file can be present in the tarball and completely unreachable to any consumer.
- **A barrel that forgets to re-export something fails as silently as possible — an undefined value, not an error.** That makes it far easier to miss in review than the export-map version of the same mistake, which at least throws.
- **Dual-package publishing doubles every subpath's surface area.** Each entry needs both module-system conditions or it breaks exactly one class of consumer while looking correct to the other.
- **Verify from a clean install of the built artifact, not from the source tree.** The gap between "shipped in dist" and "importable by a user" is invisible from inside the repository.
