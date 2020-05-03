.. title: How to setup a Windows VM to build conda packages
.. slug: how-to-setup-a-windows-vm-to-build-conda-packages
.. date: 2020-05-03 22:00:42 UTC+02:00
.. tags: python,conda,windows,epics
.. category: conda
.. link:
.. description:
.. type: text

I mostly work on macOS and Linux and I have almost no development experience on Windows.
I recently wanted to update the `epics-base feedstock <https://github.com/conda-forge/epics-base-feedstock>`_
on conda-forge_.
The goal was to have it working on the 3 platforms. A good opportunity to try building on Windows.

As explained in `conda-forge documentation <https://conda-forge.org/docs/maintainer/knowledge_base.html#particularities-on-windows>`_,
it's possible to test Windows builds even if you don't work on Windows.

Create a Windows Virtual Machine
================================

The first step is to download a Virtual Machine from https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/.

.. image:: /images/setup-windows-vm-conda/download-vm.png

I'll use VirtualBox_ as I work on macOS and already have it installed.

- Download `MSEdge.Win10.VirtualBox.zip <https://az792536.vo.msecnd.net/vms/VMBuild_20190311/VirtualBox/MSEdge/MSEdge.Win10.VirtualBox.zip>`_
- Unzip the archive
- Move the `MSEdge - Win10` directory under `~/VirtualBox VMs/`
- Open `MSEdge - Win10.ovf` to import it in VirtualBox
- Start the new VM

.. image:: /images/setup-windows-vm-conda/msedge-win10-login.png

As mentioned on the download page, the password is "Passw0rd!".

.. image:: /images/setup-windows-vm-conda/msedge-win10-home.png


Developer tools installation
============================

Now that we have a Windows VM, we need a few developers tools to build conda packages.

VScode
------

We'll first need an editor. I've been a Vim_ user for many years, but have to say I started to use VScode more lately,
with `VSCodeVim <https://github.com/VSCodeVim/Vim>`_ of course :-).
Microsoft is really doing a nice job. There are many great extensions.
I can only recommend it.

Download VScode from https://code.visualstudio.com/.

.. image:: /images/setup-windows-vm-conda/download-vscode.png

Obviously, an editor is very personal. Pick the one you prefer!

Git
---

To work with code, `Git <https://git-scm.org>`_ is essential.
Download and install it from https://git-scm.com/downloads.

.. image:: /images/setup-windows-vm-conda/download-git.png

Microsoft’s Visual C++
----------------------

To compile native code (C, C++, etc.) on Windows, we need Microsoft’s Visual C++.
As explained in this `Python wiki <https://wiki.python.org/moin/WindowsCompilers>`_,
each Python version uses a specific compiler version.

Since CPython 3.5, Visual C++ 14.X is required.
This compiler has been part of Visual Studio since Visual Studio 2015.

As of May 2020, the current version of Visual Studio that you can download from
https://visualstudio.microsoft.com/downloads/ is Visual Studio 2019,
which comes with Visual C++ 14.2.

We could use that version, but conda-forge currently uses Visual Studio 2017.
The transition from vs2015 to vs2017 was done in April 2020.
Downloading an older release requires a Microsoft account.

Once logged in, go to https://visualstudio.microsoft.com/vs/older-downloads/
and download the **Build Tools for Visual Studio 2017**.
You don't need to download the full Visual Studio edition.

.. image:: /images/setup-windows-vm-conda/download-build-tools-for-visual-studio-2017.png

During installation, only select the build tools.

.. image:: /images/setup-windows-vm-conda/install-build-tools-for-visual-studio-2017.png

The installation process will take some time. Be patient.

.. image:: /images/setup-windows-vm-conda/visual-studio-installer.png

Miniconda3
----------

Now that we have an editor, Git and Windows C++ compilers, the last tool missing is conda_.
Download and install Miniconda3 from https://docs.conda.io/en/latest/miniconda.html#windows-installers.

.. image:: /images/setup-windows-vm-conda/download-miniconda.png

To use conda, start the Anaconda Prompt from the Start menu.

.. image:: /images/setup-windows-vm-conda/start-anaconda-prompt.png

Just a few more steps to configure conda.

- Add conda-forge channel::

    conda config --add channels conda-forge

- Install conda-build::

    conda install -y conda-build

- Download the `conda_build_config.yaml` file from `conda-forge-pinning-feedstock <https://github.com/conda-forge/conda-forge-pinning-feedstock>`_
  under the home directory::

    curl -LO https://raw.githubusercontent.com/conda-forge/conda-forge-pinning-feedstock/master/recipe/conda_build_config.yaml


The `conda_build_config.yaml <https://github.com/conda-forge/conda-forge-pinning-feedstock/blob/master/recipe/conda_build_config.yaml>`_
file contains the version of compilers to use as well as the
`globally pinned packages <https://conda-forge.org/docs/maintainer/pinning_deps.html#globally-pinned-packages>`_.
Notice that the compiler is set to vs2017 for Windows.

.. image:: /images/setup-windows-vm-conda/conda-build-config-yaml.png

Note that this file contains several versions for Python: 3.6 and 3.7 at
the time of writing. This means that when building conda packages with Python, you'll always
build 2 packages (except for noarch).
You can keep it as is if you want to test every versions. In most cases, testing one version
of Python is enough. Especially during development.
You can tune that file to your needs. I'll comment out Python 3.6.

::

    python:
    #  - 3.6.* *_cpython
      - 3.7.* *_cpython

That's it! We now have all the tools required to build conda packages locally on Windows.

.. image:: /images/setup-windows-vm-conda/conda-info.png


Testing
=======

To check that everything is setup properly, let's try to build an existing conda recipe
that requires a compiler.
Start an Anaconda Prompt and run::

    mkdir conda-forge
    cd conda-forge
    git clone https://github.com/conda-forge/cython-feedstock.git
    cd cython-feedstock
    conda build recipe

The build should succeed and create the `cython-0.29.17-py37h1834ac0_0.tar.bz2` package.

.. image:: /images/setup-windows-vm-conda/cython-build.png

Summary
=======

We now have a VM with all the tools required to build and
test locally conda packages on Windows.

In a coming post, I'll detail how I built epics-base on Linux, macOS and Windows.


.. _conda-forge: https://github.com/conda-forge
.. _VirtualBox: https://www.virtualbox.org
.. _Vim: https://www.vim.org
.. _conda: https://docs.conda.io/en/latest/
