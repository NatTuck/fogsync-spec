
# FogSync Spec

## About This Document

This document describes the design of FogSync, an encrypted file
storage system to allow trees of files and directories to be securely
stored on untrusted cloud storage services.

This document is formatted in GFM in order to make it readable
directly from its github repository.

https://help.github.com/articles/github-flavored-markdown/

## Basic Concept

FogSync syncs a set of "shares". Each share is a directory tree,
kept in sync on multiple machines.

A directory tree (including files) is stored as a sequence of
"updates". Each update describes a set of changes to the
directory tree (e.g. add a file, delete a directory, etc).

Taken together, the updates describe the current state of the
tree.

Each update is a file. The updates are stored in a directory that can then be
synced with the central cloud server. The intent is that the cloud server will
only see the share, approximate timestamp, and approximate size of the update.
All other information should be obscured.

Syncing should work with a automatic cloud storage service (e.g. owncloud,
dropbox) or an explict sync tool (e.g. rsync). 

### Example

You have four computers:

 * Desktop
 * Laptop
 * Workstation
 * Phone

You have three shares, each synced on different
machines:

 * ~/Music (Desktop, Laptop, Phone)
 * ~/Email (Desktop, Laptop, Workstation, Phone)
 * ~/Documents (Desktop, Laptop, Workstation)

If we're syncing with owncloud, updates for the ~/Music share
are stored in ~/ownCloud/fog/ID-FOR-MUSIC/...

## Operation of the FogSync client.

The client uses a file watching API (e.g. inotify) to watch each file and directory
in each share. When a change occurs:

 1. Wait until the change has stablized.
 2. Add the change to the cache for the next update.

After a set amount of time, if there are changes in cache, create an update and
write it out to the updates directory. If configured to use rsync, trigger an
rsync.

Additionally, the fogsync client watches for external changes.

 * For owncloud/dropbox, use inotify on the updates directory.
 * For rsync, poll the remote server somehow.

When an external update appears, it's applied.

 * For each change in the update, if it's newer than the current version
   on the local filesystem, apply (e.g. copy out a file, delete a file, etc).

## What is an Update?

An update is a file that contains a series of changes. It is stored in the
following format:

 * Index
 * Change Data
 * Padding

The index is a table:

 * 32-bit length.
 * Entries

An index entry has the following fields:

 * Path Hash (32-byte SHA256 of relative path)
 * Change Offset (64-bit)

A change is:

 * Metadata
 * Data

Metadata is:
 * Flags (8 bytes)
   * First byte low bit is "executable?"
   * Second byte is type (0 = deleted, 1 = regular, 2 = directory, 3 = symlink)
 * Timestamp (64-bit float storing UTC Unix time)
 * Data size (64 bit)
 * Path size (8 bit)
 * Relative path

Data is the data of the file, or none for a directory.

Padding is added to the end of each update to make it a "round" size, which prevents
eavesdroppers from knowing the exact size of the changes. Padding is done such that
the goal size is reached *after* encryption and authentication - which will add
~64 bytes for nonce and MAC.

 1. Update sizes are rounded up at least to the next 16k.
 2. If 16k is less than 10% of the total update size, we round up to the power
    of two closest to 10% of the update size.

The file name of an update is the timestamp it was created at in the following format:

YYYYMMDDHHMMSSUUUUUURRRR

Where U's are microseconds, and the R's are a 16-bit random number in hex.

This allows the GC process to scan updates in order.

## Operations

There are three basic operations that can be performed on the encrypted data
store ("fog" directory):

 * Insert / update
 * Delete
 * Lookup

### Insert

On change:

 * Copy the file and metadata to a cache directory.

On commit:

 * Copy the file and metadata into an update.

### Delete

On change:
 
 * Create a tombstone (just metadata, type = deleted) in the
   cache directory.

On commit:

 * Copy the tombstone into an update.

## Lookup

 * Scan the index of *every* update for the corresponding path hash.
 * Take the one with the most recent timestamp.

## Garbage collection

When multiple changes are made to the same path across several updates, only
the most recent update is valid (until we implement delta compression). This
means the previous updates are now garbage.

Clients should occasionally scan for garbage and collect it. For a given stored
update, collecting garbage in it is a tradeoff between disk space and bandwidth.

 * If every change in an update is garbage, it can simply be deleted.
 * If some changes in an update are still live, they must be copied into
   a new update before the old update can be deleted.

Another complication: Having two machines GC the same update is a waste. It will
produce redundant copies of the same changes in multiple new updates.

 * If there are two copies of the same change (with the same timestamp), the
   copy in the *earlier* update is garbage.

Any strategy is fine, as long as:

 * The new update containing the old changes is written before the old update
   with those changes is deleted.
 * The total number of updates is kept below some maximum count. It's probably
   inefficient to have more than 10k updates at a time.

Questions:

 * Can we find out the size of a file that was deleted by seeing it removed in 
   a GC update? Should we require that GC be combined with other updates?
 * What's the best we can do on bandwidth vs. space?

# Update Encryption

Each update is encrypted before being written to the "fog" directory.

The encrypted file layout is as follows:

 * 32-byte nonce (generated randomly)
 * Data, encrypted with Twofish-CTR
 * 32-byte HMAC

