# Installation

## Clone Databases With Git
The slac-db repo uses git lfs to store its sqlite databases. You can find install isntructions for git lfs here:
https://git-lfs.com/

## Install Without DBs
A simple git clone without lfs will grab the DB code without the sqlite databases. You will have to generate the databases yourself.

## Create Databases
We have two databases at LCLS, the directory service and the Oracle database, called lcls_elements. Both are maintained by IOC and hardware engineers. We create local copies in slac-db for our own convenience. The slac-db repo combines these databases with additional metadata to create a device db.

Our three databases are:
```
lcls_elements.sqlite3
directory_service.sqlite3
device.sqlite3
```

### lcls_elements db
The lcls_elements db is sourced from a CSV file, which we pull from Oracle. This is not a long term solution to pulling from Oracle, but it's what we've been doing in development.

To pull this CSV file yourself, navigate to this website while logged into your SLAC VPN. https://oraweb.slac.stanford.edu/apex/slacprod/f?p=116:600:12734138584515:::::

Click actions, then select "download" from the drop menu.

![A Picture of the Oracle Web interface with the Download option selected.](oracle_download.png)

``` python
import slac_db.create
```