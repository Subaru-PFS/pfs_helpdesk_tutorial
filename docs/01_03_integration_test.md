## Setup environment and run integration test

After the process is complete, run the following commands to set up the environment:

```
$ source $WORKDIR/(username)/packages/stack_26_0_2/loadLSST.bash
$ setup pfs_pipe2d
$ setup -jr /$WORKDIR/(username)/packages/fluxmodeldata-ambre-20230608-full
```

(Optional) For individual users, replace the default `drp_pfs_data` package with a local version:

```
$ setup -jr $WORKDIR/(username)/packages/drp_pfs_data
```

Start the integration test in a new directory:

```
$ mkdir -p /$WORKDIR/(username)/packages/integrationTest
$ cd $WORKDIR/(username)/packages/integrationTest
$ pfs_integration_test.py -c 4 .
```

!!! note
    `-c` specifies the number of cores to be used for parallel processing.