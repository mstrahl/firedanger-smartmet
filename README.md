
# SmartMet-server for FireDanger.eu service

FireDanger information from seasonal and weather forecast data to inform Europe on risks for wildfire in their region.
This entails a GRID smartmet-server/plugins to run. It is now being tested on a EuropeanWeather cloud server running Ubuntu.
The goal is to develop lua functions for many fireweather indices, so that they can be calculated on-demand from original weather, seasonal and reanalysis data.
... let's see how it works:

First prepare a data directory at the same level as this cloned directory (../data) and `ln -s smartmet-server/config ../config`
Then you can let docker-compose build and run everything else.

# Start all services (even with --detatch the build process will wait until finished)
docker-compose up --detatch

This will quickly add all components, but below are steps for all of the three Docker containers needed.

# Transfer files from C3S CDS with shell scripts
You will need grib_set and cdo, so install something like libeccodes-tools and cdo packages on Ubuntu and equivalents on other OSs. A good option is to install ana/miniconda and install these tools with it in a dedicated conda environment. The scripts are referring to an xr environment as python xarray functionality is planned in addition to use cdo and eccodes.
Under bin you have the get-seasonal.sh script for the seasonal forecast fetching and bias adjustment postprocessing. Similar scripts for ERA5, ERA5L and a bunch of other data sets have been added as well. Check out the scripts as their internal comments explain their purpose.

These scripts are used to put data in a ~/data/grib directory, where the smartmet-server will look for new grib files to read in.

# Docker setup 
## Build and run ssl-proxy

For https connections of the server, there is an ssl-proxy image for handling this. The server name needs to be adapted in docker-compose.yaml file.

`docker-compose up --detatch --build ssl-proxy`

## Build and run postgres-database

Setup the fminames database for geonames-engine so that the smartmet-server APIs can handle place and region names in addition to coordinates.

`docker-compose up --detatch --build fminames-db`

## Build and run Redis

Setup database for storing grib-file metadata

`docker-compose up --detatch --build redis_db`

## Build and run smartmet-server

`docker-compose up --build smartmet-server`

## Fire up all four services at once

This will:

* Start the Postgresql-database and create a db-directory to store all the data there.
* Start Redis for storing information about available grib data
* Start SmartMet Server after the Postgersql is ready
* Start the ssl-proxy to enable encrypted http communication in addtion to http

`docker-compose up --detatch`

# Data ingestion and configuring on SmartMet-Server

## Read data to Redis to be used by SmartMet-server
Then docker and its four instances (smartmet-server, fminames-db, redis and ssl-proxy), put grib files with data in the ../data directory.
Filenames will have to match the pattern (dataproducer)_(YYYYMMDDTHHMMSS)_(description as you like).grib
Dataproducer needs to be something defined in the ../config/engines/grid-engine/producers.csv. For mapping data into the server refer to [DATAMAPPING.md](DATAMAPPING.md)
Mapping is in place for most of the data fetched by the scripts in bin. For data, that in the https://[yourserver]/grid-gui plugin shows up as GRIB-XXX (XXX referring to numbers), 
the mapping is not yet in place, so follow the directions to complete the mapping.

Run a `filesys-to-smartmet`-script in the smartmet-server container... once Redis is ready. The location of filesys-to-smartmet.cfg depends on where
the settings-files are located at. With `docker-compose.yaml` the settings are currently stored in `/home/smartmet/config`.

`docker exec --user smartmet smartmet-server /bin/fmi/filesys2smartmet /home/smartmet/config/libraries/tools-grid/filesys-to-smartmet.cfg 0`

This should tell you how the grib data was ingested. you can check also by going to https://[yourserver]/grid-gui

## HOPS forecasts and analysis into grid smartmet-server

### HOPS initial state and forcing data retrieval

HOPS needs initial state of soil parameters in the domain it is running for and forcing data for the forecasts. In harvester-seasons the initial state is kept
up from C3S ERA5(L) reanalysis data and the forcing is coming from C3S seasonal forecast data. Shell scripts for getting these datasets are:
`get-seasonal.sh`
`get-ERA5-daily.sh`

The scripts run without arguments to fetch the most recent available data set or can be run with year month arguments like '2020 3' for seasonal
and '2020 4 11' for daily ERA5(L) to retrieve older data. Within the shell scripts there are calls to cds-api python scripts and commands to move data to
proper directories.

To take in account of bias adjustments monthly biases are calculated with the following cmds (variables which bias was calculated have added):
* `seq 0 24 | parallel -j 16 --tmpdir tmp/ --compress cdo ymonmean -sub -selvar,2d,2t,e,tp,stl1,sd,rsn,ro, era5l/era5l_2000-2019_stats-monthly-nordic.grib -remapbil,era5l-nordic-grid -daymean -selvar,2d,2t,stl1,sd,rsn,var205 ens/ec-sf-2000_2019-stats-monthly-fcmean-{}.grib era5l-ecsf_2000-2019_monthly-fcmean-{}.grib`
* `seq 0 24 | parallel -j 16 --compress --tmpdir tmp/ cdo --eccodes div -ymonmean -selvar,tp,e era5_2000-2019_stats-monthly-euro.grib -mulc,2592000 -ymonmean -remapbil,era5-eu-grid -selvar,tprate,erate ens/ec-sf-2000_2019-stats-monthly-euro-{}.grib era5_ecsf_2000-2019_e+tp-monthly-eu-{}.grib`
* `cdo ensmean era5l-ecsf_2000-2019_monthly-fcmean-*.grib era5l-ecsf_2000-2019_monthly-bias.grib`
Using parallel makes this faster as the 16 core system can faster calculate results for 25 ensemble members than one cdo thread doing the ensemble first and then carry on.
And a mean of many biases seems to be a better idea than the bias of an ensemble mean.

The seasonal forecast can now be interpolated on the ERA5L grid and the adjustments can be added:
* `cdo ymonadd era5l-ecsf_2000-2019_monthly-fcmean-em.grib -remapbil,era5l-nordic-grid grib/ECSF_20200402T0000_all-24h-nordic.grib`
Again doing this 51 times in parallel is faster, so that's how it is done for real, but the above explain better the operation. In fact adding some timeshifting/interpolation
is needed to complete the job successfully. This was used for real, last step is needed, because cdo fails to add the ensemble attributes:
* `seq 0 50 | parallel -j 16 --compress --tmpdir tmp/ cdo ymonadd -selmonth,2020-04-02,2020-11-02 -inttime,2020-04-02,00:00:00,1days -shifttime,1year era5l-ecsf_2000-2019_monthly-bias-fixed.grib -remapbil,era5l-nordic-grid -selvar,var168,var167,var182,var205,var33,var141,var139,var228 ens/ec-sf_20200402_all-24h-nordic-{}.grib ens/ec-bsf_20200402_all-24h-nordic-{}.grib`
* `cat ens/ec-bsf_20200402_all-24h-nordic-*.grib > grib/ECBSF_20200402_all-24h-nordic.grib`
* `for f in ec-bsf_20200402_all-24h-nordic-*.grib ; do i=$(echo $f | sed s:.*nordic-::|sed s:\.grib::); grib_set -s centre=98,setLocalDefinition=1,localDefinitionNumber=15,totalNumber=51,number=$i $f ${f:0:-5}-fixed.grib; done`

As only soil temperature level 1 is available in seasonal forecasts, the deeper temperatures on level 2, 3 and 4 are prodcued by using the
ERA5L monthly statistics from 2000-2019 to give each gridpoint the relation between stl1 and the deeper temperatures. The forecasted stl1 with bias adjustement is used to produce level 2,3,4 temperatures. This data set will be used to demonstrate the added value from using HOPS.
* `seq 0 50 |parallel -j 16 --compress --tmpdir tmp/ cdo --eccodes add -seldate,2020-04-02,2020-11-02 -inttime,2020-04-02,00:00:00,1days -shifttime,1year -selvar,stl1,stl2,stl3 era5l-stls-diff-climate.grib -add -seldate,2020-04-02,2020-11-02 -inttime,2020-04-02,00:00:00,1days -shifttime,1year -selvar,stl1 era5l-ecsf_2000-2019_bias-monthly.grib -remapbil,era5l-nordic-grid -selvar,stl1 ens/ec-sf_20200402_all-24h-nordic-{}.grib ens/ec-bsf_20200402_stl-24h-nordic-{}.grib`

The EC-BSF bias adjusted data set is then used to force the HOPS model.

HOPS model operation is described in [github:fmidev/hops](https://github/fmidev/hops)

### HOPS output transformation to grib

The HOPS model produces CF conform netCDF as output that has to be turned into smartmet-server grib files under the ~/data/grib directory structure.
This is achieved by running the command 

`bin/hops-cf_to_grib.sh hops_ens_cf_2020-04-02.nc'

it uses cdo and grib_set command line commands to turn the HOPS output into grib variables and turns the Lambert-Azimtuhal-Equal-Area projection into a regular lat
lon projection over the same area.

To be available as addressable variables the grib variables need to be mapped into SmartMet-server FMI-IDs or newbase names.
A general guide explaining this is under [DATAMAPPING](DATAMAPPING.md).

# Using timeseries, WMS or WFS plugins of the SmartMet-server

The aim is to have timeseries and WMS layers for the http://harvesterseasons.com/ service and WFS downloads available for data sets that will be exported to
other service outlets of HOPS output.

Example:

`/timeseries?param=place,utctime,WindSpeedMS:ERA5:26:0:0:0&latlon=60.192059,24.945831&format=debug&source=grid&producer=ERA5&starttime=data&timesteps=5`
`/timeseries?producer=ERA5&param=WindSpeedMS&latlon=60.192059,24.94583&format=debug&source=grid&&starttime=2017-08-01T00:00:00Z`
`/wfs?request=getFeature&storedquery_id=windgustcoverage&starttime=2017-08-01T00:00:00Z&endtime=2017-08-01T00:00:00Z&source=grid&bbox=21,60,24,64&crs=EPSG:4326&limits=15,999,20,999,25,999,30–999`
`/wfs?request=getFeature&storedquery_id=pressurecoverage&starttime=2017-08-01T00:00:00Z&endtime=2017-08-01T00:00:00Z&source=grid&bbox=21,60,24,64&crs=EPSG:4326&limits=0,1000`

A big thanks to this citation for using parallel a lot:
  O. Tange (2011): GNU Parallel - The Command-Line Power Tool,
  ;login: The USENIX Magazine, February 2011:42-47.
