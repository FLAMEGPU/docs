FLAME GPU 1 Technical Report and User Guide Documentation
=======================================================

.. image:: https://readthedocs.org/projects/flamegpu/badge/?version=master
   :target: http://flamegpu.readthedocs.io/en/latest/?badge=master
   :alt: Documentation Status

This is the RST version of the FLAME GPU 1 Technical report and user guide. The documentation has been moved from latex to RST with version control via github. 

Note: For FLAME GPU 2 documentation please see [https://github.com/FLAMEGPU/FLAMEGPU2-docs](https://github.com/FLAMEGPU/FLAMEGPU2-docs).


How to Contribute
-----------------
To contribute to this documentation, fork this repository on GitHub and create a local clone of your fork on your local machine, see `Fork a Repo <https://help.github.com/articles/fork-a-repo/>`_ for the GitHub documentation on this process.

Create changes in a feature-branch and push to your fork. Please ensure that the documentation can still be built after your changes have been made (see `Building the documentation`).

Once you have made your changes and updated your Fork on GitHub you will need to `Open a Pull Request <https://help.github.com/articles/using-pull-requests/>`_. All changes to the repository should be made through Pull Requests, including those made by the people with direct push access.


Building the documentation on Linux (venv)
------------------------------------------



#. Ensure Python 3 is installed

#. Navigate to the root directory of this repository

#. Create a `virtual environment <https://docs.python.org/3/tutorial/venv.html>`_ to install sphinx and other dependencies: ::

    mkdir -m 700 ~/.venvs
    python3 -m venv ~/.venvs/flamegpu-docs

#. Activate the virtual environment. This will need repeating for future builds: ::
    
    source ~/.venvs/flamegpu-docs/bin/activate

#. Install the Python packages needed to build the HTML documentation: ::

     pip3 install -r requirements.txt

#. Build HTML the documentation: ::

     make html
  
   Or if you don't have the ``make`` utility installed on your machine then build with *sphinx* directly: ::

    sphinx-build . ./html

#. To build the PDF documentation you will need : 

   * GNU Make
   * A LaTeX distribution, such as MikTeX or TeXLive

   Then from the command line, build the pdf using: :: 

     make latexpdf
     
   Depending on your latex distribution you may need to install missing latex packages using the distribution-specific method.


Building the documentation on Windows (conda)
---------------------------------------------


#. Install Python 3 on your machine by downloading and running the `Miniconda for Python 3 <https://conda.io/miniconda.html>`_ installer: 

   * Install for *just you*;
   * Install to the default location (e.g. ``C:\Users\myusername\Miniconda3``);
   * Do **not** *add Anaconda to your PATH environment variable*;
   * Do **not** *register Anaconda as your default Python 3.6*.

#. Click *Start* -> *Anaconda3 (64-bit)* -> *Anaconda Prompt* to open a terminal window.

#. Create a new *conda environment* for building the documentation by running the following from this window: ::

    conda create --name flamegpu-docs python=3.6
    conda activate flamegpu-docs
    pip install -r requirements.txt

#. To build the HTML documentation run: ::

    make html
	
   Or if you don't have the ``make`` utility installed on your machine then build with *sphinx* directly: ::

    sphinx-build . ./html

#. If you want to build the PDF documentation you will need:

   * `GNU Make <http://gnuwin32.sourceforge.net/packages/make.htm>`_
   * `MikTeX <http://miktex.org/download>`_

   Then from the command line, the following will build the ``.pdf`` file: ::

    make latexpdf

   On first run, MikTeX will prompt you to install various extra LaTeX packages.

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
