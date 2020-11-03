- [Documentation system](#documentation-system)
  - [About the `docs` folder](#about-the-docs-folder)
  - [About the readthedocs](#about-the-readthedocs)
  - [About the language used](#about-the-language-used)
  - [About the generation/compilation](#about-the-generationcompilation)
- [Existing documents conversion](#existing-documents-conversion)
    - [Local generation](#local-generation)
    - [Online generation](#online-generation)
  - [Special files of the `docs` folder](#special-files-of-the-docs-folder)
- [Other considerations/references](#other-considerationsreferences)
  - [RegEx used for Replacements](#regex-used-for-replacements)


# Documentation system
## About the `docs` folder
The idea is to have a `docs` folder in each of the SDMX-TWG repositories where the documentation source will be stored. the `docs` folder is automatically recognized by readthedocs (more on that later) when the code is pushed back to the GitHub repository.

In that folder, the structure follows the best practices of `sphynx`. There is an `index.rst` file and a `conf.py` file that are the basis of the whole documentation system.

## About the readthedocs
There is a single user (`sdmx`) where multiple projects have been configured with links (watch) on the various branches of the corresponding repositories on Github.

On the sdmx.org domain, the DNS was changed to create a new **sub-domain** called `docs`. It has been configured to point to the [latest production release](http://readthedocs.org/project/sdmx/latest).

So far, in the readthedocs configuration, the `sdmx` project corresponds to the main documentation site and the `sdmx-ml` repository. 



## About the language used
Sphynx uses the **RST** (ReStructuredText) syntax to write the source of the documentation instead of the more simple and known MD (Markdown). It allows for fare more possibilities and more complex structure, linking, etc.

## About the generation/compilation
# Existing documents conversion
There is a well known and practical tool that will allow us to convert our existing standard word documentation into RST source files rather easily.
It is called [pandoc](htts://pandoc.org/) and can be executed in a linux or windows environment to do the majority of the work in creating the RST source files that will serve as the basis for the documentation.

These are the steps in the conversion of the Word document from the existing 2.1 standard:
1. Open the `filename.doc` in a recent version of Word
1. Save As... `filename.docx`
1. use `pandoc` to convert the `filename.docx` to `RST` 

The problem with Word documents is the way they store images... sometimes in png, jpg but often in `EMF` format... This is usually an issue when converting the word document to `RST`syntax.


### Local generation
On windows, use the `make.bat html` to generate in sub-folder `build/html`

On linux, use `make html` to genrate in sub-folder `build/html`

then open the `index.html` file found in the `build/html` folder.

Please mind that the inter-links (between repositories) will not work in the local generated content. It will instead link to the online version - *I did not find a way to re-direct to the local sibling folder easily...*

### Online generation
Using Git, the documentation can be created and version controled locally. Once the documentation is deemed to have reached a correct level and is thus ready to be published, the repository (branch) can be pushed back to the SDMX-TWG repository. This action will trigger a documentation re-compilation on readthedocs.io.

The decision was to host the documentation and generate it automatically at every commit/push on the source repositories in GitHub. Note that only some branches have been configured to be watched for changes and thus would be re-compiled and re-published.

## Special files of the `docs` folder
  * `Makefile` - Sphinx file to manage the documentation building system
  * `make.bat` - Sphinx file to manage the documentation building system
  * `sidebarGenerator.py` -  Functions for managing and generating the TOC (this files needs to be **synced** accross repositories of the SDMX-TWG)
  * `_sidebar.rst.inc` - generated file (from sidebarGenerator.py)
  * `plantuml.conf` - Common configuration for the PlantUML diagrams
  * `conf.py` - configuration of the Sphynx system & configuration of the TOC
  * `index.rst` - entry point of the documentation


# Other considerations/references
## RegEx used for Replacements 
Usage: regex to search and replace some figure captions from the automatic conversion of docx => rst using pandoc.

- `^={1,}$\n(\|.*\|)$\n^={1,}$\n(Figure .*)\n^={1,}$` to `$1 $2`
- `(\|.*\|)$\n\n(Figure .*)`to `$1 $2`




