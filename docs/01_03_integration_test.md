# Setup Environment and Run Integration Test

After the process is complete, run the following commands to set up the environment (choose the appropriate `loadLSST` script according to your shell):

```
$ source $WORKDIR/(username)/packages/stack_28/loadLSST.bash
$ setup pfs_pipe2d
```

(Optional) For individual users, replace the default `drp_pfs_data` package with a local version:
(This is only necessary if you will install curated calibs; otherwise the shared install is fine.)

```
$ setup -jr $WORKDIR/(username)/packages/drp_pfs_data
```

The integration test provides a check that the pipeline has been installed and is working properly. It is not a necessary part of the installation. It takes about half an hour to run.

Start the integration test in a new directory:

```
$ mkdir -p /$WORKDIR/(username)/packages/integrationTest
$ cd $WORKDIR/(username)/packages/integrationTest
$ pfs_integration_test.sh -c 4 .
```

!!! note
    `-c` specifies the number of cores to be used for parallel processing.
