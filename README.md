SHEDS Brook Trout Occupancy Model
=================================

Jeffrey D. Walker, PhD  
[Walker Environmental Research LLC](https://walkerenvres.com/)

Ben Letcher, PhD  
[USGS](https://www.lsc.usgs.gov/?q=cafb-research), [UMass](https://eco.umass.edu/people/faculty/letcher-ben/)

Dan Hocking, PhD  
[Frostburg State University](http://hockinglab.weebly.com/)

*Adapted from*: [Conte-Ecology/Northeast_Bkt_Occupancy](https://github.com/Conte-Ecology/Northeast_Bkt_Occupancy).

## About

This repo contains the source code for the [SHEDS Brook Trout Occupancy model](http://ecosheds.org/models/brook-trout-occupancy/latest).


## Quick Start

To set up and run the SHEDS Brook Trout Occupancy model, follow these steps:

1. Set up [Configuration File](#configuration)
2. Set version number in [Version File](#versioning)
3. Create [Working Directory](#working-directory)
4. [Run the Model](#model-execution)
5. Upload [Results](#results)
6. Update [Documentation](#documentation)

Each of these steps are described in more detail in the following sections.


## Configuration

The R scripts load a set of configuration variables from the `config.sh` file.

Because some of these variables contain sensitive information (e.g. database password) or variables that will be unique to the user's local file system (e.g. model root path), the `config.sh` file is not tracked by git.

Therefore, after cloning the repo, the user must create this file manually  by copying the template (`config.template.sh`) to `config.sh`, and then setting the appropriate values within `config.sh`.

```bash
cp config.template.sh config.sh
nano config.sh
```

The `config.sh` file must contain the following variables

```
# database connection
SHEDS_BTO_DB_HOST="<hostname>"
SHEDS_BTO_DB_PORT=<port>
SHEDS_BTO_DB_USER="username"
SHEDS_BTO_DB_PASSWORD="password"
SHEDS_BTO_DB_DBNAME="dbname"

# model root directory
SHEDS_BTO_ROOT="/path/to/bto-model-data/"

# stream temperature model version
SHEDS_BTO_STM_VERSION="1.0"
```

The last variable (`SHEDS_BTO_STM_VERSION`) corresponds to the version of the stream temperature model that should be loaded into the brook trout occupancy model from the database.


## Versioning

The model versioning approach is loosely based on [semantic versioning](https://semver.org/).

Each version contains three numbers of the form `X.Y.Z`:

- `X`: Major version incremented when there is a major change to the underlying model theory or code.
- `Y`: Minor version incremented when a new set of model inputs and outputs are created. This can either be due to an update of the input datasets or a (minor) change in the code.
- `Z`: Patch version incremented when there is a minor change to the documentation or output files, but no change to the model calibration or prediction datasets.

The full version therefore is used to track changes to both the model and the documentation. Model calibration and results do not change better minor versions (`X.Y`).

The major and minor versions are set to an environment variable called `SHEDS_BTO_VERSION` within the `version.sh` file. For example:

```
SHEDS_BTO_VERSION=1.0
```

Unlike the configuration file, the `version.sh` file is tracked by git to ensure the model version coincides with model source code. Any changes to the model code should be associated with a change to the version number.

The version can be set to any string, because it is simply used to generate the model [working directory](#working-directory). During development, for example, the version could be set to `1.0-dev`.

For official model releases, the version should use two-point semantic versioning of the form `X.Y` where `X` is the major version and `Y` is the minor version. The minor version (`Y`) should be incremented when there are only changes to the input dataset and calibration. The major version (`X`) should be incremented when there are more significant changes to the model structure or set of predictor variables.

:heavy_exclamation_mark: **Important** The environment variable `SHEDS_BTO_VERSION` to the **minor** version only, and **does not include the patch number**. In other words, it is only of the form `X.Y` (see Model Versioning above) and does not include a `v` prefix.

When a new version of the model is complete, a tagged release should be created in github with the full version of the model (`vX.Y.Z`), and a title containing both the version and the date (e.g. `vX.Y.Z (MMM DD, YYYY)`).


## Working Directory

In order to transfer data from one script to another, all of the scripts save and load data from a common directory. This directory is referred to as the model's working directory, but should **NOT** be confused with the working directory used by R (i.e. `getwd()`), which is the directory from which the scripts are run. In other words, the model's working directory stores the data files, while R's working directory contains the source scripts.

The path to the model's working directory is automatically generated by combining a model root path and the model version (i.e. `/<root path>/<version>`). The root path therefore can contain one of more sub-directories, each of which is the working directory for a specific version of the model.

The root directory should therefore look something like this:

```
$ tree ${SHEDS_BTO_ROOT}

├── 0.9
|   ├── ...
└── 1.0
    ├── bto-model.log
    ├── data-covariates.rds
    ├── data-huc.rds
    ├── data-obs.rds
    ├── data-temp.rds
    ├── model-calib.rds
    ├── model-input.rds
    ├── model-predict.csv
    ├── model-predict.rds
    └── model-valid.rds
```

The root path and model version are set using two environment variables:

- `SHEDS_BTO_ROOT`: within the `config.sh` file (see [Configuration](#configuration)), this variable should be set to a local path that serves as the root directory for all model versions (e.g. `/path/to/bto-model-data`)
- `SHEDS_BTO_VERSION`: within the `version.sh` file (see [Model Version](#versioning)), this variable should be set to a unique model version

The `load_config()` R function (defined in `r/functions.R`) will combine these two variables to create a complete path to the current model working directory, which is set to the `wd` element of the list returned from the function.

Here is an example of how these files work together:

```
# config.sh
SHEDS_BTO_ROOT="/path/to/bto-model-data"

# version.sh
SHEDS_BTO_VERSION="1.0"

# R
> source("functions.R")
> config <- load_config()
> print(config$wd)
[1] "/path/to/bto-model-data/1.0"
```

The `config$wd` path is then used throughout the various R scripts to load and save data from a single directory.

:heavy_exclamation_mark: **Important** When creating a new model version, the user must manually create the working directory on their file system. The R scripts will **NOT** automatically create this directory. The `load_config()` will return an error if it cannot find the working directory, in which case you simply need to create it and try again.


## Model Execution

The BTO model is run by executing a series of R scripts located in the `r/` directory.

These scripts comprise a chain of tasks -- each script performs one of these tasks such as fetching raw input data, merging datasets to generate a model input dataset, calibrating the model, or generating predictions.

The scripts must be run a specific order to ensure all inputs for a given script have already by generated by previous scripts. The scripts are listed in order and explained in `run-model.sh`.

In theory, one could execute `run-model.sh` to run all the R scripts sequentially after completing steps 1-3 in [Quick Start](#quick-start).

```
./run-model.sh
```

:heavy_exclamation_mark: **Remember** to change the model version within `version.sh` to create the working directory prior to running this script or you will overwrite previous results.

Or the scripts can be run individually using the `Rscript` command:

```
cd r
Rscript data-obs.R
Rscript data-huc.R
# and so on
```

Or by opening each script in RStudio and running them line by line in an interactive session. If using RStudio, open the `r/r.proj` file to load the project.

The last option is probably the most practical for new users to understand what the model scripts do exactly.


## Results

The derived metrics for each catchment are exported to:

1. database table `bto_model` (via `r/export-db.R`)
2. CSV file `r/csv/sheds-bto-model-v{VERSION}.csv` (via `r/export-csv.R`)

After completing the model run, the output CSV file should be copied to the server within the `www/static/models/bto-model/output` folder.


## Documentation

After running all of the model scripts, the model documentation should be updated.

The documentation is written using the [`bookdown` package](https://bookdown.org/yihui/bookdown/), and located within the `r/docs` sub-directory.

The home page for the documentation is written in the `index.Rmd` file. The remaining sections are each written within their own R markdown file (e.g. `01-theory.Rmd`).

The documentation can be generated from the source Rmd files using the `Build Book` button within the `Build` pane in RStudio. Alternatively, the following command can be used:

```
rmarkdown::render_site(encoding = 'UTF-8')
```

The static output files (i.e. static HTML, CSS, and JavaScript) can be found within the `r/docs/_book` sub-directory.

After the full documentation is initially generated, individual sections can be edited and re-rendered using the `Knit` button in RStudio, similar to rendering individual Rmd files.

When documentation for the current model version is complete, the static output files in `r/docs/_book` can be copied to the appropriate location on the web server using FTP, scp or some other transfer protocol.
