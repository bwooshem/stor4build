# Dynamic Charge Controls Documentation

Documentation for the controls written by LBNL (aka. "EASY-SHIFT-lite" in the conceptualization phase, although the control logic is substantially/fundamentally different from the originally EASY-SHIFT algorithm). We have erred toward overdocumenting things that might be obvious to some, to reduce the risk of confusion/mistakes.

These controls address the challenges of:
- Optimize to reduce cost under varying TOU rates, while accounting for changes in weather/load on each day
- Charging can create new demand charges by leading to spikes in demand, not good for electric tariffs with high demand charges
- Account for changes in COP of charging/discharging

## Summary of Current Development Status (as of Aug 26, 2026)

The control functions are stable in that they will run for the various models, but we are aware of several bugs that are still work in progress. __The inputs, outputs, and general structure of the control functions are expected to remain the same.__ However, the processing/logic within the controls for the exact charge/discharge schedule still needs to be improved. The only file we anticipate needing to modify is `stor4build/src/stor4build/dynamic_charge_controls.py`, which is a self-contained file for the controls functions. Thus, we recommend that __others in the project team may begin integration with the GUI tool simultaneously as we finalize the control functions.__ Expect that with some building models and weather file combinations, the schedules might not be very effective and occasionally be nonsensical at this point. The schedules will improve over time as we wrap up development.

An update in late-Aug 2026 fixed bugs where (1) the schedules sometimes did not fully optimize because a catch would cause the scheduler to exit partway through the runperiod, and (2) optimization is now by month (approximating a billing period) rather by year, improving cost savings by ~3-5\% over the previous (July 31, 2026) version.

Scope: _Should_ work for any chiller-based model that can use icetank storage. Developed primarily using LargeOffice_4A_2019.osm, tested that it runs without crashing for LargeDataCenterHighITE, LargeHotel.

Command Line - Working, tested for a few cases but not extensively

GUI - In progress.


## Command Line Usage
Command line function has been developed, loosely adapted from `run-icetank`. Refer to `src/stor4build/cli/__init__.py`. The major steps performed by the `run-icetank-dynamic` command are as follows: 
1. Run model in OpenStudio and E+ with icetank and a charge schedule that keeps the icetanks in idle the entire time to create a baseline with the same parameters we can read when scheduling the controls. This uses the new `add_pytank_with_schedule` measure, adapted from `add_pytank`. This charge schedule is a constant in `stor4build/resources/baseline_schedule_15min.csv`. The outputs of this simulation will go into a "no_icetank" folder within the output directory specified with the `-r` parameter.
2. Using the output files of step 1, run `generate_schedule()` in `stor4build/src/stor4build/dynamic_charge_controls.py`. This will run the scheduling script, which will produce a file called `dynamic_charge_schedule.csv` with TES mode and charging temperature
3. Run the model in OpenStudio and E+ again with the `add_pytank_with_schedule` measure, this time with the optimized charging mode schedule. The result will be in the `run` subdirectory of the output directory specified with the `-r` parameter.


Run using

```bash
stor4build  run-icetank-dynamic <path_to_osm> <path_to_epw> --openstudio "C:\openstudio-3.7.0\bin\openstudio.exe" -r <path_to_results_folder> -m <path_to stor4build/measures> --ntanks <int>
```

## Charge Controls Functions
The scheduling algorithm is written in Python. All of the functions are  in `stor4build/src/stor4build/dynamic_charge_controls.py`

The only function we call directly is `generate_schedule()`. This calls other functions as necessary to get the required inputs for the scheduling. 


### List of Python dependencies:
```python
import pandas as pd
import numpy as np
import datetime
from datetime import timedelta
import matplotlib.pyplot as plt
import traceback
import os
import re
import collections
from epw import epw
import sys
```

__epw__ is a package for EnergyPlus weather files (.epw). It can be installed using
```bash
pip install git+https://github.com/building-energy/epw.git@master
```

Note: it should be possible to remove `epw`. The only thing it's currently used for is OAT (drybulb). See "Required Inputs"


### Using `generate_schedule()`

Parameters
- `baseline_run_path`: os.path, to "run" folder where the results from the baseline went
- `demand_charge_schedule`: list or array, 24-hour demand charge period schedule, with different periods mapped to integers. The lowest cost period should be `0`, then the next highest `1`, `2`, etc. If not provided, will default to a sample schedule. 
- `demand_charge_rate`: list, rates for different demand periods and overall demand. The indexes map to the numbering in demand_charge_schedule, such that the lowest cost period demand charge is in index `0`, then the next highest in index `1`, etc. The length should be 1 more than the number of different demand_charge_schedule periods. The final entry, index `-1`, is an overall demand charge applied to the highest consumption regardless of time. Any of these may be 0, but all must be included for the code to work correctly. If not provided, will default to a sample tariff rate. 
- `electric_rate`: list or array, 24-hour electricity rates in $/kWh. If not provided, will default to a sample rate schedule. 
- `epw_file`: os.path, to the EnergyPlus Weather file (.epw) used to run the simulation. If none, it will default to in.epw (note: current stor4build repo does not create the in.epw, so it will likely crash if not provided)

Returns
- os.path to the resulting schedule file (.csv) containing the optimized charging schedule and charging temperature, in the format required for the add_pytank_with_schedule measure

### Required Inputs

The optimization relies on being able to obtain the baseline electric and thermal loads, chiller performance curves, chiller sizing, outdoor air temperature, and other data pulled from the various input files. This was challenging to fully automate and a common cause of bugs when trying to run with different models. We think it is working now, but if it ends in an `IndexError`, this is a probable culprit. 
- OAT: _current testing obtained it_ from .epw used to run the model, _however, it would likely be easier to add an eplusout.csv column for `'Environment:Site Outdoor Air Drybulb Temperature [C](TimeStep)'`. The `dynamic_charge_controls.py` existing code automatically looks for the aforementioned OAT column in eplusout first, then if not found, attempts to load it from the .epw file instead._ 
	- _If adding this to eplusout, if it's trivially easy, it might be worth adding relative humidity, wind speed, and solar radiation in case we decide to add these as factors to compute the schedule in a future revision, but unlikely in the near-term_
- baseline loads and performance: from eplusout.csv from the baseline run
- chiller parameters: from input .idf generated

_Note: There may be better places to get the data from. We're open to discussion and modifying the data gathering functions for the controls appropriately for ease of integration_

#### List of datapoints & columns required
1. __eplusout.csv__ columns
	1. `'Date/Time'`
	1. `"NUM TANKS:Schedule Value [](TimeStep)"` __NOTE: If we pass this in from somewhere else, we could avoid needing this in eplusout, which would allow us to use a non-TES baseline run__
	1. `"*:Chiller Electricity Rate [W](TimeStep)"`
	1. `"*:Chiller Condenser Heat Transfer Rate [W](TimeStep)"`
	1. `"*:Chiller Evaporator Cooling Rate [W](TimeStep)"`
	1. `"*:Chiller Part Load Ratio [](TimeStep)"`
	1. `"*:Chiller COP [W/W](TimeStep)"`
	1. `"Electricity:Facility [J](TimeStep)"`
	1. (Optional: `'Environment:Site Outdoor Air Drybulb Temperature [C](TimeStep)'`, if not it will go to EPW file)
1. __epw file__ Weather data: Outdoor air temperature - can be obtained from eplusout.csv if it gets added there, but it is currently being obtained from the epw file. See note on OAT under "Required Inputs" header
1. __in.idf__  (see `get_idf_info()`)
	1. `RunPeriod,` 
	1. `Curve:Biquadratic,` for `EIRFT` 
	1. `Curve:Quadratic,` for `fQRatio`  
	1. `Chiller:Electric:EIR,`
		1. Reference COP
		1. Minimum Part Load Ratio
		1. Minumum Unloading Ratio
1. __eplusout.eio__
	1. `Chiller:Electric:EIR`, `Design Size Reference Capacity [W]` for each chiller (see `get_chiller_design_capacities()`)
1. __From user arguments, will use a default if not specified__
	1. `--electric-rate`
	1. `--demand-charge-rate`
	1. `--demand-charge-schedule`
1. __passed in automatically__
	1. `--schedule_file`

## Impacts on other files/codes in the repo
1. The OSM file needs some of the `OS:Output:Variable` outputs that were removed since the old s4b repo, as these are inputs to the dynamic charge controls. We added them back to some of the `.osm` files. See the "Add Output:Variable to osm files for dynamic charge controls" commit. 
2. The `run-icetank-dynamic` function is added to `src/stor4build/cli/__init__.py`. Appropriate entries are also added to the --help output
3. The older s4b repo would save "in.epw" with the input weather file. The current version does not. We were originally assuming we could use the in.epw file, but built another method that accepts the path to the epw file automatically rather than modifying the existing code.

To our knowledge, the additions/changes do not impact the functioning of anything else in the repo.

## Brainstorming on integration for GUI
Here's a rough idea of what we think needs to be added (not necessarily the best way to do this, but mentioning in case it's useful)
- Toggle button to activate dynamic charge controls, with checks to ensure it only is selectable for icetank
- Inputs for electricity rate & demand charge rate and on the backend, a way to convert it to the format used in the controls (see "Using `generate_schedule()`" section)
- Remove inputs for charge/discharge start/end
- We assume there's already a way to select bldg type, storage size, and location for weather

## Caveats/known bugs and quirks
__Timestep__: Currently, the code only works with a timestep of 15 minutes (or 4 timesteps per hour). This is enforced in the `add_pytank_with_schedule` measure. _LBNL is working on refactoring the code to handle any timestep between 1 minute and 1 hour, treating as low priority for now unless others request it sooner._

__Controls Schedule Issues:__ During testing, periods of up to 1 month in the summer led to fairly effective schedules. However, during the full integration, we observed that runperiods that include the full year often have strange results, particularly large chunks of time in "discharge" mode during the non-cooling season, with no "charge" periods to offset it. We are investigating several possible causes, all of which involve small tweaks to the controls logic. Fixing this should not impact the integration with the overall tool. 
- Update: changing the code to break the initial optimization to minimize costs by month instead of the entire runperiod improved the overall cost savings by about 3-5\%. Further testing is required to identify if shorter optimization windows or other modifications can yield larger improvements. 

__Python Warnings__: There are a few FutureWarnings that pop up when running the code. We fixed most of them, but a few are still lurking. We tested with Python 3.13 and 3.8. If others using newer versions get an actual error due to these, let us know and we'll figure it out. 

__Expanding to other TES__: In _theory_, these controls should work for any TES. We'd need to change the input parameters and constraints significantly to handle other cases (and the data frameworks), then test and validate the controls. This will have to be left for future work.

__Modifying/Recompiling__: if anything is changed in the Python scripts, first need to run this command to reflect the changes when running stor4build commands. 

```bash
pip uninstall stor4build -y && pip install . 
```

Changes to measure.rb scripts do not require this.