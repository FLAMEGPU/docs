.. image:: https://readthedocs.org/projects/flamegpu/badge/?version=master
:target: http://flamegpu.readthedocs.io/en/latest/?badge=master
:alt: Documentation Status
    
FLAME GPU Technical Report and User Guide Documentation
=======================================================
This is the RST version of the FLAME GPU Technical report and user guide. The documentation has been moved from latex to RST with version control via github. 


How to Contribute
-----------------
To contribute to this documentation, fork this repository on GitHub and create a local clone of your fork on your local machine, see `Fork a Repo <https://help.github.com/articles/fork-a-repo/>`_ for the GitHub documentation on this process.

Create changes in a feature-branch and push to your fork. Please ensure that the documentation can still be built after your changes have been made (see `Building the documentation`).

Once you have made your changes and updated your Fork on GitHub you will need to `Open a Pull Request <https://help.github.com/articles/using-pull-requests/>`_. All changes to the repository should be made through Pull Requests, including those made by the people with direct push access.


Building the documentation
--------------------------

#. Install Python on your machine 

#. Navigate to the root directory of this repository

#. Install the Python packages needed to build the HTML documentation: ::

    pip3 install -r requirements.txt

#. To build the HTML documentation run: ::

    make html
  
   Or if you don't have the ``make`` utility installed on your machine then build with *sphinx* directly: ::

    sphinx-build . ./html

#. To build the PDF documentation you will need : 

    * GNU Make
    * A LaTeX distribution, such as MikTeX or TeXLive

   Then from the command line, build the pdf using: :: 

     make latexpdf
     
   Depending on your latex distribution you may need to install missing latex packages using the distribution-specific method.

Continuous build and serve
--------------------------

The package `sphinx-autobuild <https://github.com/GaretJax/sphinx-autobuild>`_ provides a watcher that automatically rebuilds the site as files are modified. To use it, install (in addition to the Sphinx packages) with the following: ::

    pip install sphinx-autobuild

To start the autobuild process, run: ::

    sphinx-autobuild . ./html

The application also serves up the site at port ``8000`` by default at http://localhost:8000.


Making Changes to the Documentation
-----------------------------------

The documentation consists of a series of `reStructured Text <http://sphinx-doc.org/rest.html>`_ files which have the ``.rst`` extension. These files are then automatically converted to HTMl and combined into the web version of the documentation by sphinx. It is important that when editing the files the syntax of the rst files is followed. 


If there are any errors in your changes the build will fail and the documentation  will not update, you can test your build locally by running ``make html``. The easiest way to learn what files should look like is to read the ``rst`` files already in the repository.

Submitting Changes and Making Contributions
-------------------------------------------

Contributions should be made by forking the documentation site repo (this repo) and submitting a pull request. Pull requests will be merged by an Admin after review. 
