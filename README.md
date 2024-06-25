# PostEddyPro Vignette
PostEddyPro R package usage for carbon flux data analysis post Eddypro preprocessing.

# How to use PostEddyPro package for gapfilling fluxes?
download this repository by clicking on the following link: https://github.com/bravemaster3/PostEddyPro_Temp_Vignette/archive/refs/heads/main.zip

Unzip it. Then, open the PostEddyPro_Temp_Vignette.Rproj file, which should open a project in your Rstudio.
The advantage of opening a project is that yit automatically sets your working directory to that project folder.
You can also see files in the files tab (bottom right panel...).

Click on the PostEddyPro_Usage.Rmd file from the file tab, or open from the folder.
Then, you will have things similar to what is below...

You can copy your own dataset formated like the one named sample_df_oneyear.csv, into the data subfolder. So, after this, you will have it along with sample_df_oneyear.csv in the same "data" folder.

Then, remember to replace the line where it is read by your own file

# How to use PostEddyPro package for gapfilling fluxes?

## Installing packages
```{r}
install.packages("devtools")
devtools::install_github("bravemaster3/PostEddyPro")
```

## Loading the required libraries and your dataset
```{r echo = FALSE}
library(PostEddyPro)
library(data.table)
library(dplyr)
library(parallel)
library(foreach)
df <- fread("data/sample_df_100_days.csv", header = TRUE)
#VIEW DF TO ENSURE IT LOOKS GOOD, ESPECIALLY THE timestamp column
```

## Running the gapfilling
!!!Important to note that all the input parameters should not have gaps
This means that they have been gapfilled prior to this step.
```{r echo=FALSE}
#If you want to try different hyper parameters instead, you can pass a list to tuning_grid argument. Read the help of xgboost_gapfiller by copying and running ?xgboost_gapfiller in your console
gf_list <- xgboost_gapfiller(
  #the dataframe
  site_df=df,
  #time stamp column
  datetime="timestamp",
  #name of the flux column already quality checked and filtered 
  flux_col="NEE",
  #List of predictors for the XGBoost model. Note that "yearly_sin", "yearly_cos" and "delta" are calculated internally by the xgboost_gapfiller function
  preds=c("Ta", "Ts", "SWin", "SWout", "VPD", "RH", "yearly_sin", "yearly_cos","delta"),
  sitename="Halsingfors" #This variable is simply for annotating the plots with the site name.
)
```

Here are the hyperparameters that were tuned:
vectors represent the default values.

nrounds = c(500, 1000, 1500)
max_depth = c(5,10, 15)
eta = c(0.05, 0.3)
gamma = c(0, 0.5)
colsample_bytree = c(0.65, 0.75)
min_child_weight = c(2, 5)
subsample = c(0.50.75)
  
Read more about the meaning of each hyper parameter here: https://www.hackerearth.com/practice/machine-learning/machine-learning-algorithms/beginners-tutorial-on-xgboost-parameter-tuning-r/tutorial/

## Exploring the list output from the previous gapfilling
```{r echo=FALSE}
gf_list$tuningmodel$bestTune #the best hyperparameters
View(gf_list$site_df) #Opens the final dataframe containing NEE_filled (the final NEE to be used). !!!column "predicted" is not from crossvalidation
df_cv <- gf_list$pred_meas$data 
View(df_cv) #Opens the cross validation table. NEE_filled here is the prediction of the holdout set in all folds of the 10-fold cross validation
gf_list$r_sq_cv #is the R squared from cross validation
gf_list$rmse_cv #is the RMSE from cross validation.
#You can calculated the mean absolute bias yourself if needed using the df_cv table

#You can save this list as an RDS object
saveRDS(gf_list, "data/gf_rds.RDS")#change the path if you prefer
#You can read a saved RDS object as follows
gf_list <- readRDS("data/gf_rds.RDS")

#if instead you want to save the dataframe, you can do so like this:
fwrite(gf_list$site_df, "data/gapfilled_df.csv", dateTimeAs = "write.csv")
```

## Formatting fluxes for ReddyProc partitioning

```{r echo=FALSE}
formatting_fluxes_REddyProc(df=gf_list$site_df,
                            datetime = "timestamp",
                            flux_col = "NEE_filled",
                            FLUX="NEE",#this you should not change when working on NEE.
                            rg_col = "SWin",
                            tair_col = "Ta",
                            tsoil_col = "Ts",
                            rh_col = "RH",
                            vpd_col = "VPD",
                            ustar_col = "u_star",
                            saving_path = "data",#this is a path that you like. In this case, data is a folder if you opened the project PostEddyPro_Temp_Vignette. Because it sets automatically that folder as working directory
                            filename = "For_REddyproc")#it appends automatically a .txt extension
```

## partitionning
```{r}
reddyproc_gapfiller(formatted_file_path="data/For_REddyproc.txt",
                    saving_folder="data",#this is the folder path
                    FLUX="NEE",
                    gapfill_flux = FALSE)
```

## Montecarlo simulation

```{r}
root_folder = "data" #you can change this to another path if you wish
monte_dir = file.path(root_folder, "Montecarlo")
if(dir.exists(monte_dir)) unlink(monte_dir, recursive = TRUE)
if(!dir.exists(monte_dir)) dir.create(monte_dir)

mc_sim_hals_list <- montecarlo_sim(df_gf = gf_list$site_df, #the gapfilled flux dataframe. This would be retrieved for instance from the rf_gapfiller function
                                   df_cv = df_cv,
                                   flux_col="NEE",
                                   flux_pred_col="NEE_filled", #including all predictions also for the measured data
                                   datetime = "timestamp",
                                   preds = c("Ta", "Ts", "SWin", "SWout", "VPD", "RH", "yearly_sin", "yearly_cos","delta"),
                                   n = 100,
                                   saving_folder = monte_dir,
                                   flux_sign = "both"
)

```

## Explore the formula used for simulation
```{r}


mc_sim_hals_list$slope_pos
mc_sim_hals_list$intercept_pos

mc_sim_hals_list$slope_neg
mc_sim_hals_list$intercept_neg

#You can uncomment the following if you want to see what dataset is used to do the calculations...
#mc_sim_hals_list$df_cv_sum_Week

```

## Gapfilling each Montecarlo simulation using the best tuned hyperparameters

```{r}

monte_dir_gf = file.path(root_folder, "Montecarlo_gf")
if(dir.exists(monte_dir_gf)) unlink(monte_dir_gf, recursive = TRUE)
if(!dir.exists(monte_dir_gf)) dir.create(monte_dir_gf)

montecarlo_sim_gf_CO2_xgb(mc_sim_path = monte_dir,
                          best_tune = gf_list$tuningmodel$bestTune, #this is the best tuning during the caret crossvalidation, should be fetched from the saved RF gapfilling model prior to using this function
                          preds = c("Ta", "Ts", "SWin", "SWout", "VPD", "RH", "yearly_sin", "yearly_cos","delta"), #same as used in the original gapfilling
                          flux_col = "NEE",
                          mc_sim_gf_path = monte_dir_gf,
                          datetime = "timestamp")
```


## Partitioning each Montecarlo simulation. Trying to make use of more cores

```{r}
root_folder = "data" #you can change this to another path if you wish
format_monte_dir = file.path(root_folder, "Montecarlo_format")
if(dir.exists(format_monte_dir)) unlink(format_monte_dir, recursive = TRUE)
if(!dir.exists(format_monte_dir)) dir.create(format_monte_dir)

monte_dir_pt = file.path(root_folder, "Montecarlo_pt")#this is the path where partitioned will go
if(dir.exists(monte_dir_pt)) unlink(monte_dir_pt, recursive = TRUE)
if(!dir.exists(monte_dir_pt)) dir.create(monte_dir_pt)

#BE careful with these... whenever you registerDoparallel, you need to stop the cluster. Otherwise, you will have memory leak!!
no_cores = parallel::detectCores()-1
#!!
cl <- parallel::makePSOCKcluster(no_cores)
doParallel::registerDoParallel(cl)
#!!

all_gf <- list.files(monte_dir_gf, full.names = TRUE)

`%dopar%` = foreach::`%dopar%`
useless_output = foreach::foreach(file = all_gf, .packages = c("PostEddyPro", "data.table", "dplyr")) %dopar% {
  df <- data.table::fread(file)
  df_compl <- gf_list$site_df %>% select("timestamp", "LE", "H", "u_star") %>% dplyr::left_join(df, by="timestamp")

  formatting_fluxes_REddyProc(df=df_compl,
                              flux_col = "NEE_filled",
                              FLUX="NEE",
                              saving_path = format_monte_dir,
                              filename = paste0("For_ReddyProc_",sub('\\.csv$', '',basename(file))),
                                                          rg_col = "SWin",
                            tair_col = "Ta",
                            tsoil_col = "Ts",
                            rh_col = "RH",
                            vpd_col = "VPD",
                            ustar_col = "u_star",
                              datetime = "timestamp")
}
#!!
parallel::stopCluster(cl)
foreach::registerDoSEQ()
#!!

#The previous was just formatting, and this one will do partitioning
montecarlo_sim_noCH4_gf(mc_sim_path = format_monte_dir,
                          mc_sim_gf_path = monte_dir_pt,
                          flux_col="NEE",
                          gapfill_flux = FALSE)


```

## Merge partitioned montecarlo carlo simulations

```{r}
df_mc <- merge_montecarlo_sims(gf_type="REddyProc",
                               dir = monte_dir_pt,
                               saving_dir = "data", #you can change this to another directory if you wish
                               filled_flux_col1 = c("NEE_f","GPP_f", "Reco"))
```

There is more to PostEddyPro package, and I will try to write more tutorials later on...
