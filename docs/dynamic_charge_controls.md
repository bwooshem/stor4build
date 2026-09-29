# Dynamic Charge Controls Reference

The Dynamic Charge Controls purpose is to minimize cooling electricity costs under TOU rates that may include energy and/or demand charge components. Schedules vary each day, depending on baseline load, rate, and the expected COP under different conditions.

These controls address the challenges of:
- Optimize to reduce cost under varying TOU rates, while accounting for changes in weather/load on each day.
- Charging can create new demand charges by leading to spikes in demand, not good for electric tariffs with high demand charges. Adjusting both TES operating mode and charging temperature can modulate these.
- Account for changes in COP between charge, idle, and discharge modes; and with variation in outdoor air temperature. 

Dynamic Charge Controls are currently implemented and tested for icetank TES. 

## Usage:

```bash
stor4build  run-icetank-dynamic OSM EPW --openstudio "C:\openstudio-3.7.0\bin\openstudio.exe" -r <path_to_results_folder> -m <path_to stor4build/measures> --ntanks <int>
```

### Options

__`--openstudio`__ `<openstudio>`  
	OpenStudio CLI to use.  
		__Default__  
			`'openstudio'`  

__`--r`__, __`--run-dir`__ `<run_dir>`  
	Directory to run in.  
		__Default__  
			`'.'`

__`--m`__, __`--measures-dir`__ `<measures_dir>`  
	Directory containing measures.  
		__Default__  
			`'.'`

__`--n`__, __`--ntanks`__ `<N>`  
	Number of tanks.  
		__Default__  
			`1`

<!-- NOTE: Many of the other options for the regular `run-icetank` will probably still work. Those can be copy-pasted here. -->   

### Arguments

__OSM__
	Required argument. Path to `.osm` file.

__EPW__
	Required argument. Path to `.epw` file.


## Dynamic Charge Control Algorithm

### Adjusting to Current Load

### Consideration of COP

COP is obtained using the chiller curves defined in the `.idf` or `.osm` file. Two performance curves are accounted for:
1. "EIRFT" Curve:Biquadratic for chiller performance relative to outdoor air temperature.
2. "fQRatio" Curve:Quadratic for chiller performance based on part load ratio.

The calculations also require finding the `cop_ref`, `min_plr` and `min_ur` for the Chiller:Electric:EIR.

### Temperature Control


## Simulation Workflow

This section describes the major steps performed by the `run-icetank-dynamic` command: 
1. Run model in OpenStudio with the icetanks in idle the entire time to create a baseline with the same parameters we can read when scheduling the controls. This uses the `add_pytank_with_schedule` measure, adapted from `add_pytank`. This charge schedule is a constant in `stor4build/resources/baseline_schedule_15min.csv`. The outputs of this simulation will go into a "no_icetank" folder within the output directory specified with the `-r` parameter.
2. Using the output files of step 1, run `generate_schedule()` in `stor4build/src/stor4build/dynamic_charge_controls.py`. This will run the scheduling script, which will produce a file called `dynamic_charge_schedule.csv` with TES mode and charging temperature. The process is described in the [Dynamic Charge Control Algorithm](#dynamic-charge-control-algorithm) section.
3. Run the model in OpenStudio again with the `add_pytank_with_schedule` measure, this time with the optimized charging mode schedule. The result will be in the `run` subdirectory of the output directory specified with the `-r` parameter.


## Limitations

__Currently, only Icetank TES is supported__: In _theory_, these controls should work for any TES. However, this would require significant modification of the input parameters and constraints to handle each case. Analysis would be required to validate the controls work as intended for these new cases.

__Timestep must be 15 minutes__: Currently, the code only works with a timestep of 15 minutes (or 4 timesteps per hour). This is enforced in the `add_pytank_with_schedule` measure. _This is planned to be fixed in a future release_

__COP calculations assume all chillers are identical__: The function to [automatically detect the COP](#consideration-of-cop) uses a string search in the original simulation file. 

## Extending Controls

The dynamic charge controls functions in `src/stor4build/dynamic_charge_controls.py` can be modified in a local installation by advanced users. Potential use cases are to make small modifications to the control logic to better handle a different rate structure or building type, or using the existing processing code to communicate as a framework to test other controls strategies.

Recompiling is necessary to reflect any modifications to the the Python scripts when running `stor4build` commands. Run the following after modifications are complete, but before running any `stor4build` command.

```bash
pip uninstall stor4build -y && pip install . 
```

Note this is not required for changes to Ruby scripts.
