# PostEddyPro Vignette
PostEddyPro R package usage for carbon flux data analysis post Eddypro preprocessing.

# How to use PostEddyPro package for gapfilling fluxes?

Download this repository by clicking on the following link: https://github.com/bravemaster3/PostEddyPro_Temp_Vignette/archive/refs/heads/main.zip

Unzip it. Then, open the PostEddyPro_Temp_Vignette.Rproj file, which should open a project in your Rstudio.

The advantage of opening a project is that it automatically sets your working directory to that project folder.
You can also see files in the files tab (bottom right panel...).

Open the Rmd file inside CH4_gapfilling folder, named "gapfilling_FCH4.Rmd" from the file tab, or open from the folder in your explorer.

Then, you will have things similar to what is below...

You can copy your own dataset formatted like the one named EC_data_qc_filtered.csv, into the data subfolder (.../PostEddyPro_Temp_Vignette/CH4_gapfilling/data/EC_data_qc_filtered.csv), that should already be quality checked. So, after this, you will have it along with EC_data_qc_filtered.csv in the same "data" folder.

Then, remember to replace the line where it is read by your own file, and also make sure the predictors you use are in your dataset, except yearly_cos, yearly_sin, and delta, which are calculated from the datetime column internally to the function.

## Steps:

1. **Install packages** - Install devtools and PostEddyPro from GitHub
2. **Load libraries and data** - Load required packages and read your quality-checked eddy covariance data
3. **Run gapfilling** - Use `rf_gapfiller_fast()` function with your data and predictors
4. **Explore results** - View cross-validation metrics, plots, and the gapfilled dataset
5. **Do montecarlo simulation** - if needed, proceed to do montecarlo simulation, or do it with your own scripts.

**Important**: All input predictor variables must be gapfilled separately (not included in this guide) before running the Random Forest gapfilling step.

See the [gapfilling_FCH4.Rmd](gapfilling_FCH4.Rmd) file for the complete workflow with executable code.