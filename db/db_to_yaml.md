# Writing YAML Files Using DB

This database is based on the [slicops device db](https://github.com/slaclab/slicops/blob/main/slicops/device_sql_db.py). Their db is fully integrated with a different device object than we have in `slac-devices`. The copy of this database in `slac-db` doesn't have the same integration, but is instead written to support the existing YAML devices. The module `slac_db.db_to_yaml` handles this conversion. The hope is, in the near future, both types of devices will have the same information. This will facilitate a transition to the slicops device.

The following instructions are for including new devices in YAML files from the device db. This does not require access to the production system, unlike the original method of translation. This tutorial assumes you have already created a device in `slac-devices` using the pydantic device model.

## TLDR

1. Add your desired device type to this dictionary. The key is the keyword of the device in Oracle, and the value is the user friendly name you give your device. e.g. `map["BEND"] = magnet` [here](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/db_to_yaml.py#L5)

2. Add the expected PV address suffixes to this YAML [here](https://github.com/slaclab/slac-db/blob/main/slac_db/package_data/accessor_names.yaml). e.g.
```
PROF:
    IMAGE: image
```

3. Add metadata to this YAML [here](https://github.com/slaclab/slac-db/blob/main/slac_db/package_data/wire_metadata.yaml). e.g.
```
OTRDG02:
    meta: value
```

4. Test your new yaml by running `slac_db.db_to_yaml.get_device`. e.g.
``` python
import slac_db.db_to_yaml
slac_db.db_to_yaml.get_device("OTRDG02")
```

5. Rebuild all YAMLs with:
``` python
import slac_db
slac_db.db_to_yaml.build()
```

## Generate Device YAML with Device DB

### 1. Find Your Device in Oracle

The device db is a sqlite database that combines information from the Oracle Database and Directory Service. The `db_to_yaml.py` uses the local sqlite dbs to create the YAML files used in `slac-devices`. In order to create a new device YAML, it must first be present in the Oracle database. You can check the Oracle database by following this link while logged into the SLAC network:

https://oraweb.slac.stanford.edu/apex/slacprod/f?p=116:600:13048186753130

![A screengrab of the SLACPROD LCLS-Elements web interface for queries. This query has four clauses: Active = 'A'; Beampath contains 'SC_DIAG0'; Type='MAD'; Keyword contains 'PROF'. The query matches four results with element names: 'YAG01LEI'; 'YAG01B'; OTRDG02; OTRDG04.](oracle_query.png)

What the Oracle db calls a "keyword" is the device's type. We organize these types into a more readable and granular typing called "YAML type". Here's a mapping of Oracle Type to YAML Type for all of our current devices at the time of writing ([Source](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/db_to_yaml.py#L5)):
``` python
_ORACLE_TO_YAML_TYPE_MAP = {
    "SOLE": "magnets",
    "QUAD": "magnets",
    "XCOR": "magnets",
    "YCOR": "magnets",
    "BEND": "magnets",
    "PROF": "screens",
    "WIRE": "wires",
    "LBLM": "lblms",
    "BPM": "bpms",
    "LCAV": "tcavs",
    "INST": "pmts"
}
```
The function [_build_types](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/db_to_yaml.py#L94) in `slac_devices.db_to_yaml` uses this dictionary to pull the desired types from the local oracle db.

Occasionally, the control system and Oracle are not 1-1, as is the case for Klystrons. We are working on a solution for these devices, and they are unavailable in the meantime.

### 2. List Expected Addresses

The directory service contains about a million PVs, but HLA developers interface with far fewer than that. In addition, the PV addresses aren't convenient for programming or interpreting device behavior. [Slicops](https://github.com/slaclab/slicops/blob/fee7b5098880c1f3587797164e283989cdc5d068/slicops/device/__init__.py#L101) generalized this concept as an "accessor", which contains an address and a developer friendly name.

Our YAML files already used this pattern. For instance, profile monitors have the following confiugration in [accessor_names.yaml](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/package_data/accessor_names.yaml#L10)
``` yaml
# Oracle name of device
PROF:
    # Format:
    # PV_SUFFIX: dev_name
    Image:ArraySize0_RBV: n_row
    Image:ArraySize1_RBV: n_col
    N_OF_COL: n_col
    N_OF_ROW: n_row
```
The name `n_row` is easier to interpret than `Image:ArraySize0_RBV`, and it combines concepts accross cameras with different addresses, here `N_OF_COL`.

Choose what PVs you want to include in the device db, and add them to [accessor_names.yaml](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/package_data/accessor_names.yaml#L10).

### 3. Add Metadata

There are a lot of columns in the Oracle db, but they are not comprehensive and not every column is included in the Oracle db. If you want to add a column the lcls_elements db from the online Oracle db, you will have to modify the (Oracle schema definition)[https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/oracle.py#L181].

If you want to add metadata outside of the Oracle database, you will have to add to our [metadata file](https://github.com/slaclab/slac-db/blob/main/slac_db/package_data/wire_metadata.yaml):
``` yaml
IN20:
  detectors:
  - PMTINJ03:DL1
  - PMTINJ05:DL1
  - PMT21350:LI21
  default_detector: PMTINJ03:DL1
```
This file supports two data types, strings and YAML style object dumps. The object dumps can be recovered by running `yaml.safeload(db_value)`. We are currently working on more complete integration with this feature.

### 4. Test Yout Output

Run the following to make sure your new output can initialize a YAML type device.

``` python
import slac_db.db_to_yaml
import slac_db.your_device.Example # Device you created
device = slac_db.your_device.Example(
    slac_db.db_to_yaml.get_device("EXDEV0") # Name of a device you want to test
)
```

### 5. Create New YAML Files

Once you are satisfied that your db generation works, you can recreate all the YAML files with the new devices.

``` python
import slac_db
slac_db.db_to_yaml.build()
```