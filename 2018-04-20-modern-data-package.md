I just pushed up a [little package called **cites**](https://github.com/ecohealthalliance/cites), which provides a convenient way to access a full snapshot of the [CITES wildlife trade database](https://trade.cites.org/). This package incorporates several practices I've been using for internal packages at [my job](https://www.ecohealthalliance.org/) that I thought would be good to demonstrate.

This package design pattern is designed for sharing data with certain characteristics

# Why data packages at all?

-  As a person who thinks a lot about scientific reproducibility, I am concerned about versioning, long-term archives, cross-compatibility.
-  Data packages can go against this:
   -  Stored in software distribution hubs, not data archives
   -  Language-specific data formats

So why use them?
-   Ease access to data
-   Provide tools to smooth data analysis process along with data, _for the assumed or targeted use cases_
-   So we make assumptions of how people use the data,

# Data package design patterns

-  Classic: data in the package
   -  Nearly half of packages on CRAN have data
       -  rOpenSci unconf report
   -  Simple infrastructure
   -  Package size limits (CRAN 5MB, GitHub 100MB)
   -  Data is static (at least with version of package)
-  API: lots of packages, lots of them rOpenSci
   -  Stabilizing design pattern
   -  Wrap the API but provide convenince for conversion, etc.  Otherwise its just httr calls
      -  Authentication
      -  Conversion to R types, often data frames
      -  Pagination
      -  Data is live - can get the latest, but also results can change unexpectedly
      -  Versioning, reproducibility must be implemented in the API, or analyst saves data locally, breaking ecohealthalliance
      -  Not great for bulk data - need many/large calls, often done repeatedly

-   Data sync
    -   Better for bulk data if its small enough to handle locally
    -   More efficient for local analysis
    -   Versioning considerations can be built into the package
    -   Examples, eidith, cites, taxizedb

-   File type considerations
    -   SQLite: SQL back-end common.  Row, not column focused, can store JSON
        -   MonetDB, fast and column-based, not a single file
    -   fst: great, fast as hell, single table, not disk-based
    -   rds: general R data types, not disk-based
    -   Apache arrow - here it comes!
    -


-   Are updated at some moderate pace but not high-frequency, so that named/tagged releases are appropriate
    -   But for which backwards-compatibility and version control are important
-   One expects to analyze in bulk, so that frequent API calls to a central database for large data extracts are slow/wasteful
-   Are small enough to store locally, but large enough that download time and RAM may be a consideration.

Examples: taxizedb


Data versioning and releases with **datastorr**
===============================================

[**datastorr**](https://github.com/ropenscilabs/datastorr) by @richfitz is a great package designed for versioning and releasing medium-sized data on github.

Recently Rich has been doing some major refacotring to make **datastorr** work with (1) private repositories and (2) out-of-memory data sources. You'll find these changes on the [a development branch](https://github.com/ropenscilabs/datastorr/tree/i14-private-repos) which is what I'm pointing my `Remote:` dependency to.

**datastorr** is designed to be extended. With the right hooks, it could be made to work with other versioned sources besides github releases. such as a Zenodo archive or a Dat repository. Perhaps you wnant to contribute!

What's more **datastorrr** with auto-generate code for you to include in your package.

Efficient random access storage with **fst**
============================================

`.fst` is a binary storage format for R data frames with some great features. The [**fst** package](https://github.com/fstpackage/fst) reads and write blazingly fast, has optional compression with not much loss of performance, it stores dates properly, and it provides random on-disk access. This last property allowed Kirill Müller to make a [**dbplyr** interface](https://github.com/krlmlr/fstplyr) letting users use (some) **dplyr** verbs to query the data on-disk.

This all makes `.fst` a great choice for distributing the data in my data packages. Compression means the file transfer is relatively fast, and one can fit some pretty big data in GitHub 2GB size limit for release files. By making on-disk access the default data-reading function in my package, users can inspect and subset data without loading it all into memory. We frequently collaborate with users with older computers and spotty internet connections, so this resource efficiency is always a consideration.

Some of my internal packages that share more complex data do so by populating an SQLite database on download. SQLite (or other options for on-disk dbs, like [MonetDB](https://github.com/hannesmuehleisen/MonetDBLite-R)), gives you the ability to do more operations on-disk before pulling data into memory. It may be a better choice for multi-table databases where joins are a big part of the expected workflow with the data. This is the strategy use in Zebulun Arendsee and @sckott's [**taxizedb**](https://github.com/ropensci/taxizedb) which provides local SQLite versions of taxonomic databases that [**taxize**](https://github.com/ropensci/taxize) accesses via web APIs.

Metadata as manipulatable, searchable data.
===========================================

The CITES database provides field descriptions and a codebook in a PDF document on it's website. But *metadata is data* and should be provided in manipulatable ways. So all metadata are provided as tables returned by functions in the package such as `cites_codes()`. What's more, I've used the [**DT** package](https://rstudio.github.io/DT/) to provide searchable versions of these tables embedded in the package help files:

The hacky code for this in [my previous post](https://discuss.ropensci.org/t/searchable-metadata-in-help-files-with-htmlwidgets/1078). It has [some limitations](https://github.com/ecohealthalliance/cites/issues/2).

([Thomas Leeper's **tabulizer** package](https://github.com/ropensci/tabulizer) was invaluable for extracting this information from the PDF.)

Data processing alerts
========================

-  Sometimes you want to give users some extra nudges when working with data.  Our field data can get pretty messy, there are a lot of unique situations that may arise that we have to consider in analyses.



Other bonus evolving best practices.:
=====================================

-   As I work for an nonprofit but we have a lot of private data sets to deal with, I test my packages on CircleCI, which provides a nice free tier. It's quite easy to set up; [Marek Rogala has written a how-to](https://appsilondatascience.com/blog/rstats/2018/02/07/circleci.html).
-   To provide minimum installation friction for non-CRAN packages, I tell users to install with Gabor Csardi's [`install-github.me` service](https://github.com/r-lib/remotes#installation), which lets users install a package from GitHub without installing **remotes** or **devtools** first, just by running `source("https://install-github.me/<USER>/<REPO>")`. (Works fine with private repos)
-   I generate a `codemeta.json` file, which provides helpful, cross-language, machine readable data on the package itself, with @cboettig's [**codemetar** package](https://github.com/ropensci/codemetar).
-   I discovered today that if you codify your inclusive development practices with a `CODE_OF_CONDUCT.md` file, GitHub shows a handy pop-up about it to new issue filers. `usethis::use_code_of_conduct()` provides a handy template. Of course, one should only add such a code if you [think carefully about what it means and how you are going to enforce it and foster an inclusive community in your project](https://www.contributor-covenant.org/#enforcing-the-contributor-covenant).

