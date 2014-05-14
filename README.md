blueflower
==========

![logo](blueflower.jpg)

blueflower is a command-line tool that looks for secrets such as private keys
or passwords in a file structure.
Interesting files are detected using heuristics on their names and on
their content.

Unlike some forensics tools, blueflower does not search in RAM, and
does not attempt to identify cryptographic keys or algorithms in
binaries.

**DISCLAIMER:** This program is under development. It may not work as
expected and it may destroy your computer. Use at your own risk.


Features
------------

* *search* in the following types of files:
    - `text/*` MIME-typed files
    - PDF, DOCX, XLSX documents
    - tar, ZIP archives
    - bzip2, gzip compressed files/archives
* *detection* of 
    - common key and password containers (SSH id\_\* , Apple
      keychain, Java KeyStore, etc.) 
    - common encrypted containers (Truecrypt, PGP Disks, GnuPG files,
      encrypted ZIPs, etc.)
    - other interesting files (Bitcoin wallets, PGP policies, etc.)
* *hiding* of secrets searched for (names, secret keys, etc.) via a hash
  file


Usage
------------

### Execution

From the project's top directory, you can directly run
```
python blueflower.py directory [hashes]
```
where

* `directory` is the root of the file structure to explore
* `hashes` is an optional file, which should be created with the script
[`makehashes.py`](makehashes.py) (see details below)

Results are written to a log file `blueflower-YYYYMMDDhhmmss` in CSV format.

(Run `make clean` if you wish to remove previous log files as well as
`.pyc`'s.)

**WARNINGS:**

* no limit is set on the number of files processed (`^C` to gracefully interrupt)
* RAR archives nested in other archives are not supported 
* there may be a lot of false positives


### Installation

To install to the global packages directory:
```
sudo make install
```
(omit `sudo` on Windows)

To install locally (to site.USER_BASE):
```
make local
```

(Run `make cleanall` if you wish to clean up the project's directory.)

blueflower can then be called from any location, assuming that the
binaries are located in a directory included in your PATH.


### Python-less execution on Windows

To build a package executable on Windows machines that don't have Python
installed, do the following (on a Windows machine with Python
installed):

1. install [py2exe](http://www.py2exe.org)
2. in blueflower's `setup.py`, add `import py2exe`, remove the line
```
    version=blueflower.__version__,
```
and add the line
```
    console=['blueflower/__main__.py'],
```
3. run `python setup.py py2exe`

This will create an executable and associated files in the `dist/` folder.

Make sure that the external modules are installed as
unzipped eggs rather than as `.egg` files (`easy\_install` option
`--always-unzip`).





Hashes file
-----------

Let's say you have a list of strings that you want to search for without
revealing them. These could be names, passwords, secret keys, etc.

blueflower implements this feature, by taking as optional argument a
list of hashes.
Obviously this comes with a performance penalty: hashing all strings
matching the regular expression given.
 

### Creation 

First, put your secret strings in a text file with one item per line,
for example

```
banana1
banana2
banana3
```

Then, run 
```
python makehashes.py yourfile
```

which will prompt you for
* a regular expression that matches the set of strings
* a password, which will be needed to run blueflower

This will create a file `yourfile.hashes` in the same directory as
`yourfile`, to give as a second argument to blueflower.


### Format

The first line of `.hashes` files contains the regular expression.

The second line contains 2 lowercase hex strings separated by a comma
(no space). These are generated by [`makehashes.py`](makehashes.py), and
are respectively 

* a *salt* of 8 bytes (16 hex characters), generated using
  `os.urandom(8)`
* a *verifier* of 8 bytes, which is the SipHash-2-2 hash of the salt
  using the key derived from the password

The verifier serves to ensure that the password entered is the correct
one, by checking that hashing the salt using the password entered yields
a value identical to the verifier.

The subsequent lines include the SipHash-2-2 hashes of each of the
secret strings, in the same order as received, using the verified key.

For example, the first 5 lines of a `.hashes` file can be 

```
\b[a-z]{2,20}\b
694d63f4630c6617,d478449c1a95f8c0
06f12ba4c3b57a9f
91f5ed23a2cc21ac
381cd35a5d203fea
```

### Security 

#### Key derivation

A 128-bit key is derived from the password using SipHash-1000-100000 (the
[SipHash](https://131002.net/siphash) PRF with 1000 compression rounds
and 100000 finalization rounds).
Evaluating SipHash-1000-100000 takes approximately two seconds on an AMD
FX-8150 at 3.6GHz.
It should be slow enough to mitigate bruteforce attacks, and the use of
a salt makes precomputation useless.

SipHash-1000-100000 was chosen rather than a dedicated password hashing
scheme (bcrypt/scrypt/PBKDF2) to preserve simplicity, minimize
dependencies, and because the GPU-friendliness of SipHash can be
compensated by really slow hashing.

The hashing speed is mostly independent of the password's length, since
the 100000-iteration bottleneck is the finalization.


#### Verifier and salt

The presence of the verifier string allows to efficiently test the
correctness of a key, and thus to bruteforce keys at a high rate by
computing many SipHash-2-2 in parallel.
However keys are 128-bit, and thus practically unbreakable.
Again, the use of a salt makes precomputation useless.

The use of a same salt for both key derivation and verifier generation
might look surprising, but it does not reduce security since different
hash functions are used, and the unpredictability property is not
affected.


#### Regular expression 

The choice of the regular expression leaks information on the secrets
strings searched for, so choose it carefully: the more general, the less
leak, but the slower too (more strings will be hashed).


#### Log file

The log file only contains the hash corresponding to the secret string
detected, not the string matched. 
The log file thus does not directly reveal the secrets searched for.
However, 

* the log file contains the name of the file including the secret string
* one can easily modify blueflower to include the secrets detected in
  the log file



Dependencies
------------

Python modules:
* [pyPdf](https://pypi.python.org/pypi/pyPdf/)
* [xlrd](https://pypi.python.org/pypi/xlrd/)


Intellectual property
---------------------

blueflower is copyright (c) 2014 Jean-Philippe Aumasson, and under
[GPLv3](LICENSE).

The [`siphash.py`](blueflower/utils/siphash.py) module is copyright (c)
2013 Philipp Jovanovic, and under MIT license.

The drawing in the image [`blueflower.jpg`](blueflower.jpg) is copyright
(c) 2014 Melina Aumasson, and under [CC BY-NC-ND
3.0](http://creativecommons.org/licenses/by-nc-nd/3.0/).
