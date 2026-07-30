# Writing YAML Files Using DB

## TLDR

1. Add your desired device type to this dictionary. The key is the keyword of the device in Oracle, and the value is the user friendly name you give your device. e.g. `map["BEND"] = magnet` [here](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/db_to_yaml.py#L5)

2. Add the expected PV address suffixes to this YAML. e.g.
```
PROF:
    IMAGE: image
```
[here](https://github.com/slaclab/slac-db/blob/main/slac_db/package_data/accessor_names.yaml)

3. Add metadata to this YAML. e.g.
```
OTRDG02:
    meta: value
```
[here](https://github.com/slaclab/slac-db/blob/main/slac_db/package_data/wire_metadata.yaml)

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

The device db is a sqlite database that combines information from the Oracle Database and Directory Service. The `db_to_yaml.py` uses the local sqlite dbs to create the YAML files used in `slac-devices`. In order to create a new device YAML, it must first be present in the Oracle database. You can check the Oracle database by following this link while logged into the SLAC network:

https://oraweb.slac.stanford.edu/apex/slacprod/f?p=116:600:13048186753130

![A screengrab of the SLACPROD LCLS-Elements web interface for queries. This query has four clauses: Active = 'A'; Beampath contains 'SC_DIAG0'; Type='MAD'; Keyword contains 'PROF'. The query matches four results with element names: 'YAG01LEI'; 'YAG01B'; OTRDG02; OTRDG04.](oracle_query.png)

What the Oracle db calls a "keyword" is the device's type. We organize these types into a more readable and granular typing called "YAML type". Here's a mapping of Oracle Type to YAML Type for all of our current devices at the time of writing:

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
[Source](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/db_to_yaml.py#L5)

The function [_build_types](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/db_to_yaml.py#L94) in `slac_devices.db_to_yaml` uses this dictionary to pull the desired types from the local oracle db.

[Expected Address Suffixes](https://github.com/slaclab/slac-db/blob/main/slac_db/package_data/accessor_names.yaml)
[Metadata](https://github.com/slaclab/slac-db/blob/main/slac_db/package_data/wire_metadata.yaml)

## Add to YAML
[Where Extractors Get Called](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/write.py#L33)
[Where Extractors Live](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/generate.py#L303)
[Where Extractors Live](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/controls_information.py)
[Where Extractors Live](https://github.com/slaclab/slac-db/blob/5377feaafd2dd0ee460fe440af6a40ad3e518763/slac_db/metadata.py)

