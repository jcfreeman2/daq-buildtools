# Getting started

## System requirements

To get set up, you'll need access to the ups product area /cvmfs/dune.opensciencegrid.org/dunedaq/DUNE/products, as is the case, e.g., on the lxplus machines at CERN. 

## Setup `daq-buildtools`
`daq-buildtools` is a simple package that provides environment and building utilities for the DAQ Suite. 
Each cloned daq-buildtools can serve as many work areas as the developer wishes. 

The repository can simply be cloned via
```bash
git clone https://github.com/DUNE-DAQ/daq-buildtools.git -b v2.0.0
```
This step doesn't have to be run more than once per daq-buildtools version. 

When using The `dbt` setup script has to be sourced to make the `dbt` scripts available in the commandline regardless of the current work directory. Run:
```bash
source daq-buildtools/dbtsetup-env.sh
```
..and you'll see something like:
```
Added /your/path/to/daq-buildtools/bin to PATH
Added /your/path/to/daq-buildtools/scripts to PATH
DBT setuptools loaded
```
After this step `daq-buildtools` scripts and aliases will be accessible from your commandline regardless of the current working directory. They include `dbt-create.sh`, `dbt-build.sh` and the aliases `dbt-setup-build-environment` and `dbt-setup-runtime-environment` which are used in the following sections.

## Creating a development area (AKA work area)

To get set up, you'll need access to the ups product area `/cvmfs/dune.opensciencegrid.org/dunedaq/DUNE/products`, as is the case, e.g., on the lxplus machines at CERN. If you're on a system which has access to this product area, simply do the following after you've logged in to your system and created an empty directory (we'll refer to it as "MyTopDir" on this wiki):
```sh
dbt-create.sh <daq release name> # e.g. dunedaq-v2.0.1
```
which will set up an area to place the repos you wish to build.

```txt
MyTopDir/
├── dbt-settings
├── build/
├── log/
└── sourcecode/
    └── CMakeLists.txt
    └── dbt-build-order.cmake
```

The list of known releases is shown by 
```
dbt-create.sh --list
```

If you look in `./sourcecode` you'll see there's a repo already downloaded, `./sourcecode/daq-cmake`. This will be needed in order to build any other standard DUNE DAQ packages. For the purposes of instruction - as well as the fact that it's very likely that your own package will depend on it - let's also build the appfwk package. Downloading the appfwk is simple:
```
cd sourcecode
git clone https://github.com/DUNE-DAQ/cmdlib.git -b develop
git clone https://github.com/DUNE-DAQ/appfwk.git -b develop
cd ..
```
Note that in those commands we not only downloaded the `appfwk` repo but also the `cmdlib` repo; this is because `appfwk` depends on `cmdlib`, and this package isn't (yet, Oct-15-2020) available via ups. 

## Adding extra UPS products and product pools

Sometimes it is necessary to tweak the baseline list of UPS products or even UPS product pools to add extra dependencies dependencies. 
This can be easily done by editing the `dbt-settings` file generated by `dbt-create.sh` and adding the new entries to `dune_products_dirs`  and `dune_products` as needed. 

```bash
dune_products_dirs=(
    "/cvmfs/dune.opensciencegrid.org/dunedaq/DUNE/products" 
)

dune_products=(
    "cmake v3_17_2"
    # "gdb v9_2"
    "gcc v8_2_0 e19:prof"
    "boost v1_70_0 e19:prof"
    "cetlib v3_10_00 e19:prof"
    "TRACE v3_15_09"
    "folly v2020_05_25 e19:prof"
    "ers v0_26_00c e19:prof"
    "nlohmann_json v3_9_0b e19:prof"
    "ninja v1_10_0"
    "pistache v2020_10_07 e19:prof"
)

dune_python_version="v3_8_3b"
```
Note: depend on the release version, the actual package version may be different than those listed above.

As the names suggest, `dune_products_dirs`, `dune_products` containes the list of UPS product pools and UPS products sourced by `dbt-setup-build-environment`. Don't forget running `dbt-setup-build-environment` in a new shell 
after editing the `.dunedaq_area` file.


## Compiling
We're about to build and install `daq-cmake`, the `cmdlib` package which depends on it, and the `appfwk` package which in turn depends on `cmdlib`. By default, the scripts will create a subdirectory of MyTopDir called `./install `and install the packages there. If you wish to install them in another location, you'll want to set the environment variable `DBT_INSTALL_DIR` to the desired installation path before taking any further action.

Now, do the following:
```sh
dbt-setup-build-environment  # Only needs to be done once in a given shell
dbt-build.sh --install
```
...and this will build `daq-cmake`, cmdlib and appfwk in the local `./build` subdirectory and then install them as packages either in the local `./install` subdirectory or in whatever you pointed `DBT_INSTALL_DIR` to. 

To work with more repos, add them to the `./sourcecode` subdirectory as we did with appfwk. Be aware, though: if you're developing a new repo which itself depends on another new repo, `daq-buildtools` may not already know about this dependency. "New" in this context means "not found on https://github.com/DUNE-DAQ as of Oct-15-2020". If this is the case, you have one of two options:

* (Recommended) Add the names of your new packages to the `build_order` list found near the bottom of `./sourcecode/CMakeLists.txt`, placing them in the list in the relative order in which you want them to be built. 
* First clone, build and install your new base repo, and THEN clone, build and install your other new repo which depends on your new base repo. 

`dbt-build.sh` will by default skip CMake's config+generate stages and go straight to the build stage _unless_ either the `CMakeCache.txt` file isn't found in `./build` or you've just added a new repo to `./sourcecode`. If you want to remove all the contents of `./build` and run config+generate+build, all you need to do is add the `--clean` option, i.e.
```
dbt-build.sh --clean --install
```
And if, after the build, you want to run the unit tests, just add the `--unittest` option. Note that it can be used with or without `--clean`, so, e.g.:
```
dbt-build.sh --clean --install --unittest  # Blow away the contents of ./build, run config+generate+build, and then run the unit tests
```
..where in the above case, you blow away the contents of `./build`,  run config+generate+build, install the result in `./install` and then run the unit tests.

To check for deviations from the coding rules described in the [DUNE C++ Style Guide](https://github.com/DUNE-DAQ/styleguide/blob/develop/dune-daq-cppguide.md), run with the `--lint` option:
```
dbt-build.sh --lint
```
...though be aware that some guideline violations (e.g., having a function which tries to do unrelated things) can't be picked up by the automated linter. 

If you want to see verbose output from the compiler, all you need to do is add the `--verbose` option:
```
dbt-build.sh --verbose 
```

You can see all the options listed if you run the script with the `--help` command, i.e.
```
dbt-build.sh --help
```
Finally, note that both the output of your builds and your unit tests are logged to files in the `./log` subdirectory. These files may have ASCII color codes which make them difficult to read with some tools; `less -R <logfile name>`, however, will display the colors and not the codes themselves. 

</details>

## Running

In order to run the applications built during the above procedure, the system needs to be instructed on where to look for the libraries that can be used to instantiate objects. This is handled by the `dbt-setup-runtime-environment` script which was placed in MyTopDir when you ran dbt-create.sh; all you need to do is the following:
```
dbt-setup-runtime-environment
```

Note that if you add a new repo to your development area, after building your new code you'll need to run the script again. 

Once the runtime environment is set, just run the application you need.  

We're now going to go through a demo in which we'll send vectors from appfwk's FakeDataProducer DAQ module to its FakeDataConsumer DAQ module. The package that interface (cmdlib) the modules with the command sending functionality, is already cloned. It comes with a basic implementation that is capable of sending available command objects from a pre-loaded file, by typing their command IDs to standard input. This command facility is useful for local, test oriented use-cases. In the same runtime area, launch the application like this:
```
daq_application -c stdin://sourcecode/appfwk/schema/fdpc-job.json
```

For a more realistic use-case, where you can send commands to the application from other services and applications, the [restcmd](https://github.com/DUNE-DAQ/restcmd) library provides a command handling implementation through HTTP. To use this plugin, we'll want to obtain and build it. Since restcmd relies on the pistache package which isn't automatically set up in `dbt-setup-build-environment`, we'll want to first set up pistache:
```sh
cd sourcecode
git clone https://github.com/DUNE-DAQ/restcmd.git -b develop
cd ..
dbt-build.sh --install
```
And now let's start up daq_application:
```sh
dbt-setup-runtime-environment   # Needed to pick up restcmd libraries and applications
daq_application --commandFacility rest://localhost:12345
```
To control it, let's open up a second terminal, set up the environment, and start sending it commands:
```sh
cd MyTopDir
dbt-setup-runtime-environment
python ./sourcecode/restcmd/scripts/send-restcmd.py --interactive --file ./sourcecode/restcmd/test/fdpc-commands.json
```
You'll now see
```txt
Target url: http://localhost:12345/command
This is a list of commands.
Interactive mode. Type the ID of the next command to send, or type 'end' to finish.

Available commands: [u'init', u'conf', u'start', u'stop']
Press enter a command to send next: 
```
And what you want to first is type `"init"`. The surrounding quotes are needed, otherwise you're exited out. You can look in the other terminal running daq_application to see it responding to the commands. Next, type `"conf"` to execute the configuration, and then `"start"` to begin the actual process of moving vectors between the two modules. You should see output like the following:
```log
2020-Nov-06 13:22:51,876 DEBUG_0 [dunedaq::appfwk::FakeDataProducerDAQModule::do_work(...) at /home/jcfree/daqbuild_xcheck_instructions/sourcecode/appfwk/test/plugins/FakeDataProducerDAQModule.cpp:118] Produced vector 272 with contents {-1, 0, 1, 2, 3, 4, 5, 6, 7, 8} and size 10 DAQModule: fdp
2020-Nov-06 13:22:52,860 DEBUG_0 [dunedaq::appfwk::FakeDataConsumerDAQModule::do_work(...) at /home/jcfree/daqbuild_xcheck_instructions/sourcecode/appfwk/test/plugins/FakeDataConsumerDAQModule.cpp:122] Received vector 272: {-1, 0, 1, 2, 3, 4, 5, 6, 7, 8} DAQModule: fdc
```
To stop this, send the "stop" command. Ctrl-c will exit you out of these applications. 

<details><summary>daq_application Command Line Arguments</summary>

Use `daq_application --help` to see all of the possible options:
```sh
bash$ daq_application --help
daq_application known arguments (additional arguments will be stored and passed on):
  -c [ --commandFacility ] arg CommandFacility URI
  -h [ --help ]                produce help message
```

</details>

<details><summary>Some additional information</summary>

### TRACE Messages

To enable the sending of TRACE messages to a memory buffer, you can set one of several TRACE environmental variables _before_ running `appfwk/apps/simple_test_app`.  One example is to use a command like `export TRACE_NAME=TRACE`.  (For more details, please see the [TRACE package documentation](https://cdcvs.fnal.gov/redmine/projects/trace/wiki/Wiki). For example, the [Circular Memory Buffer](https://cdcvs.fnal.gov/redmine/projects/trace/wiki/Circular_Memory_Buffer) section in the TRACE Quick Start talks about the env vars that you can use to enable tracing.)

To view the TRACE messages in the memory buffer, you can use the following additional steps:

* [if not done already] `export SPACK_ROOT=<your spack root> ; source $SPACK_ROOT/setup-env.sh`
* [if not done already] `spack load trace`
* `trace_cntl show` or `trace_cntl show | trace_delta -ct 1` (The latter displays the timestamps in human-readable format.  Note that the messages are listed in reverse chronological order in both cases.)

</details>
