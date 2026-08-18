# Setting up Butler for DP2

```{abstract}
This tech note describes how to setup a LSST Butler at iDAC to be used with the Rubin Data Preview 2.
```

## Prepare a Postgres DB

Setup a Postgres DB with a recent version of Postgres such as 17.5. `btree_gist`
extension needs to be activated using the following command

```
CREATE EXTENSION IF NOT EXISTS btree_gist;
```

By default, butler will look at `$HOME/.lsst/postgres-credentials.txt` for 
credentials to access the database. This file should be owned by the user running
the butler command, or accessing butler via python, should have permission
600 or 400, and should have line like:

```
192.168.0.100:5432:lsstdb1:my_db_account:my_db_account_credential
```

## The file layout of DP2 data

Under the iDAC storage path were DP2 data are transferred (we refer this directory
as \<rootDir\>), there should be at least two directories: `dp2`, `raw`.

- `dp2` is where most of the DP2 products are stored
  - `dp2/etc/e-dp2-db-export.tgz` is a tar ball of parquet files, exported from the
     EDP2 database at the USDF. 
- `raw` contains a directory `refcats`, and in the future, other data

`dp2` and `refcats` are the two main categories of data in the EDP2.


## Import Butler DB

Untar the above `e-dp2-db-export.tgz` to a local storage. We will refer this directory
as \<dp2-export\>.

The following command will use `butler_transfer` to import those parquet files to
the butler database. At this point, DO NOT set up the LSST pipelines stack – it will
interfere with the `uv` package manager used by `butler_transform`.

```
# Install the package manager used by butler_transform
pip install uv
# Check out the butler_transform repository
git clone https://github.com/lsst-dm/butler_transform.git
cd butler_transform
# This will be 1.1.0 instead tickets/DM-55636 after the PR merges down
git checkout 1.1.0
```

There are two ways to ingest the parquet files to the butler. They are related to how
file paths are stored in the butler database.

### Relative path ingestion

This method will store to the database with file paths relative to your butler repo.
Alse see the `buter.yaml` below. To use this method, run the following command:

```
uv run python -m lsst.butler_transform.releases.dp2.import_dp2 --mini <dp2-export> \
    schema_name postgresql://192.168.0.100:5432/lsstdb1
```

- `--mini`: Set up repository for the “early DP2” release instead of “full” DP2 release.
- `schema_name`: Postgres schema name to use for the imported Butler repository. The
   schema name will be used in the `butler.yaml` file when creating the butler repo.
   Avoid using hyphen (`-`) in the schema name.

### Ingest with storage absolute path

This method will store to the database with absolute file paths. To use it, you will
first need to extend the `butler_tranform` by adding a path mapping function. An
example of this by the FrDF can be found at
https://github.com/lsst-dm/butler_transform/pull/22/changes.

Using the above example from the FrDF (so the mapping parameter in CLI is `frdf`)

```
uv run python -m lsst.butler_transform.releases.dp2.import_dp2 --mini <dp2-export> \
    --file-map frdf \
    schema_name postgresql://192.168.0.100:5432/lsstdb1
```

## Create Butler Repo

The next step is to create a `<rootDir>/butler.yaml` file. If you are using relative
path ingestion, create a file like this:

```
datastore:
  cls: lsst.daf.butler.datastores.chainedDatastore.ChainedDatastore
  datastores:
    - cls: lsst.daf.butler.datastores.fileDatastore.FileDatastore
      #name: dp2
      records:
        table: dp2_datastore_records
      root: <butlerRoot>/dp2
    - cls: lsst.daf.butler.datastores.fileDatastore.FileDatastore
      #name: refcats
      records:
        table: refcats_datastore_records
      root: <butlerRoot>/raw/refcats
registry:
  db: postgresql://192.168.0.100:5432/lsstdb1
  managers:
    attributes: lsst.daf.butler.registry.attributes.DefaultButlerAttributeManager
    collections: lsst.daf.butler.registry.collections.synthIntKey.SynthIntKeyCollectionManager
    datasets: lsst.daf.butler.registry.datasets.byDimensions.ByDimensionsDatasetRecordStorageManagerUUID
    datastores: lsst.daf.butler.registry.bridge.monolithic.MonolithicDatastoreRegistryBridgeManager
    dimensions: lsst.daf.butler.registry.dimensions.static.StaticDimensionRecordStorageManager
    opaque: lsst.daf.butler.registry.opaque.ByNameOpaqueTableStorageManager
  namespace: schema_name
```

If you create the butler database using absolute path, create `butler.yaml` like
this:

```
datastore:
  cls: lsst.daf.butler.datastores.fileDatastore.FileDatastore
  name: dp2
  records:
    table: dp2_datastore_records
  root: <butlerRoot>
registry:
  db: postgresql://192.168.0.100:5432/lsstdb1
  managers:
    attributes: lsst.daf.butler.registry.attributes.DefaultButlerAttributeManager
    collections: lsst.daf.butler.registry.collections.synthIntKey.SynthIntKeyCollectionManager
    datasets: lsst.daf.butler.registry.datasets.byDimensions.ByDimensionsDatasetRecordStorageManagerUUID
    datastores: lsst.daf.butler.registry.bridge.monolithic.MonolithicDatastoreRegistryBridgeManager
    dimensions: lsst.daf.butler.registry.dimensions.static.StaticDimensionRecordStorageManager
    opaque: lsst.daf.butler.registry.opaque.ByNameOpaqueTableStorageManager
  namespace: dp2
```

If your `butler.yaml` is stored in a webdav store, you will need to setup environment
variable `LSST_RESOURCES_WEBDAV_CONFIG`, pointing to a yaml config file. For more
detail, refers to https://pipelines.lsst.io/modules/lsst.resources/dav.html
