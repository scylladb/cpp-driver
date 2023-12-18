# Building the docs

## Prerequisites

To build the documentation of this project, you need a UNIX-based operating system. Windows is not fully supported as it does not support symlinks.

You also need the following software installed `doxygen` to generate the reference documentation of the driver. You can install it using the following command:

**Ubuntu**

```bash
sudo apt-get install doxygen
```

**MacOS**

```bash
brew install doxygen
```

Once you have installed the above software, you can build and preview the documentation by following the steps outlined in the [Quickstart guide](https://sphinx-theme.scylladb.com/stable/getting-started/quickstart.html).

## Custom commands

To generate the reference documentation of the driver, run the command `make javadoc`. This command generates the reference documentation using the Javadoc tool in the `_build/dirhtml/<VERSION>/api` directory.
