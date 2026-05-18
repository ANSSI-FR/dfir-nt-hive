
# DFIR-NT-HIVE

[![badge_repo](https://img.shields.io/badge/ANSSI--FR-DFIR--Ogre-white)](https://github.com/ANSSI-FR/dfir-ogre)
[![category_badge_external](https://img.shields.io/badge/category-external-%23b556b6)](https://anssi-fr.github.io/README.en.html#project-categories)
[![openess_badge_A](https://img.shields.io/badge/code.gouv.fr-collaborative-blue)](https://anssi-fr.github.io/README.en.html#openness-level)


This project is part of the [DFIR-OGRE](https://github.com/ANSSI-FR/dfir-ogre) suite and provides an interface for accessing keys, values, and data stored in *hive* files.
Hive files can be found in `C:\Windows\system32\config` and store what is commonly called the *Windows registry*.
This crate supports the hive format that is used from Windows NT 4.0 up to the current Windows 11. 

It is a fork of the [nt-hive](https://github.com/ColinFinck/nt-hive) project to add some forensic capabilities.

## Features
* Support for reading keys, values, and data from any byte slice containing hive data (i.e. anything that implements [`zerocopy::SplitByteSlice`](https://docs.rs/zerocopy/0.3.0/zerocopy/trait.ByteSlice.html)).
* Basic in-memory modifications of hive data (as [required for a bootloader](https://github.com/reactos/reactos/pull/1883)).
* Iterators for keys and values to enable writing idiomatic Rust code.
* Functions to find a specific subkey, subkey path, or value as efficient as possible (taking advantage of binary search for keys).
* Error propagation through a custom `NtHiveError` type that implements `Display`.
  As a bootloader may hit corrupted hive files at some point, nt-hive outputs precise errors everywhere that refer to the faulty data byte.
* Full functionality even in a `no_std` environment (with `alloc`, some limitations without `alloc`).
* Static borrow checking everywhere. No mutexes or runtime borrowing.
* Zero-copy data representations wherever possible.
* No usage of `unsafe` anywhere. Checked arithmetic where needed.
* Platform and endian independence.
* (dfir-nt-hive) give access to the timestamp and security descriptors of the nodes.
* (dfir-nt-hive) relax the way data is read to allow retrieval of nodes that don't follow the specifications correctly 

Check out the [docs](https://docs.rs/nt-hive), the tests, for more ideas how to use nt-hive.

## French Cybersecurity Agency (ANSSI)
<img src="https://www.sgdsn.gouv.fr/files/styles/ds_image_paragraphe/public/files/Notre_Organisation/logo_anssi.png" alt="ANSSI logo" width="25%">

*This projet is managed by [ANSSI](https://cyber.gouv.fr/). To find out more, you can visit the [page](https://cyber.gouv.fr/enjeux-technologiques/open-source/) (in French) dedicated to ANSSI’s open-source strategy. You can also click on the badges above to learn more about their meaning.*
