# UNFOLDing-Robustness-Plasticity-Evolvability-and-Canalisation-of-Biological-Function

This repository contains code to reproduce the work from the manuscript "A Unified Framework To Dissect Robustness Plasticity Evolvability and Canalisation of Biological Function." (https://www.biorxiv.org/content/10.1101/2025.03.03.641119v1.full.pdf) \ \
**Workflow Diagram**\
![workflow_diagram](https://github.com/user-attachments/assets/bcd67d8a-d91f-4365-a751-c789bc7d2553)

Follow these steps to reproduce the results:
## Data Generation by Simulation

### Step 1: Generate Prerequisites
Run `code/data_generation/prerequisites_to_simulation.py` to create:
- Adjacency matrix file with 16,038 networks (in `/data/common/` folder)
- A file with 10,000 parameter sets using Latin Hypercube Sampling (LHS), indexed 0-9999
- Initial condition guesses using LHS (for finding steady states with 'fsolve' in Step 3)
- A Python file containing ODE models as functions for all 16,038 networks

### Step 2: Sample Networks
Run `/data_generation/sample_networks_for_analysis.py` to:
- Create 10 partitions of the 16,038 networks
- Each partition contains 10% of all networks but preserves the edge probabilities of the complete set of 16038 networks
- Generates files named `lhs_models_sampled_for_analysis<sampled_dataset_id>.csv` (with <sampled_dataset_id> from 0-9) in `/data/common/`

### Step 3: Calculate Steady States
Run `code/data_generation/get_steady_states.py` to:
- Use 'fsolve' with the initial guesses to find steady states for each network across all parameter sets
- Generate one file per network (16,038 total) containing:
  - Steady-state concentrations for all three nodes (columns x[0], x[1], x[2])
  - Parameter set index (column param_index)
  - A 'steady_state_found' flag

### Step 4: Generate Time Course Dataset
Run `code/data_generation/main_dataset_generation.py` to:
- Use parameters, ODE models, and steady states as initial conditions
- Run for networks specified in the `lhs_models_sampled_for_analysis<sampled_dataset_id>.csv` files
- Output files go to `/data/v<version_id>/dataset<sampled_dataset_id>_lhs/` 
- In our case, `<version_id>` ranges from 0-2 (three sets of 10,000 parameter sets)
- Running this for all `<sampled_dataset_id>` values (0-9) completes the simulation for all 16038 networks
- Outputs include:
  - Concentration time course data in `model<model_id>_output_conc.csv`
  - Parameter values and initial conditions in `/input_sim_data/dataset_model<model_id>.csv`

## Computational Pipeline

### Iteration 0

1. **Cluster Time Series**
   - Run `code/pipeline/cluster_time_series_handling_nan.py`
   - Takes time course data from `/data/v<version_id>/dataset<sampled_dataset_id>_lhs/model<model_id>_output_conc.csv`
   - Clusters the time courses for each model
   - Outputs cluster labels to `/data/csvs/sampled_dataset<sampled_dataset_id>/cluster_labels/cluster_label_model<model_id>.csv`

2. **Calculate Cluster Barycenters**
   - Run `code/pipeline/cluster_berycenter_dataset.py` with `barycenter_flag = 0`
   - Drops clusters with fewer than 10 time courses
   - Creates file `/data/v<version_id>/csvs/sampled_dataset<sampled_dataset_id>/barycenter_dataset/barycenter<iteration_number>_sampled_dataset<sampled_dataset_id>.csv`
   - This file contains barycenters of all clusters from all networks in the sample

### Iteration 1 Onwards

1. **Cluster Barycenters**
   - Run `code/pipeline/cluster_barycenter_dataset.py`
   - Uses the barycenter dataset from the previous iteration as input (instead of simulation-generated time courses)
   - The output file `fun_cluster_labels<iteration_number>_barycenter<iteration_number>_sampled_dataset<sampled_dataset_id>.csv` is created in the folder `data/v<version_id>/csvs/sampled_dataset<sampled_dataset_id>/functional_cluster_labels`

2. **Calculate New Barycenters**
   - Run `code/pipeline/cluster_berycenter_dataset.py` with `barycenter_flag = 1`
   - The output file `barycenter<iteration_number + 1>_sampled_dataset<sampled_datast_id>.csv` is created in the `data/v<version_id>/csvs/sampled_dataset<sampled_dataset_id>/barycenter_dataset` folder

### When to Stop Iterations
- After each iteration, plot the barycenters
- Stop when time courses are visually distinct

### Final Steps & Last Pipeline Run
- After completing the pipeline iterations for all 10 partitions of networks, i.e. <sampled_dataset_id> from 0-9 for a given `<version_id>`:
  - Run `code/pipeline/merge_barycenter_datasets.py` to merge the 10 barycenter datasets and save the `barycenter<iteration_number>_v<version_id>_combined0_9.csv` file in the `data/v<version_id>/csvs/combined0_9` folder
  - Rerun the pipeline with the merged barycenter dataset `barycenter<iteration_number>_v<version_id>_combined0_9.csv` as input
    For this:
    - Change paths in `code/pipeline/cluster_barycenter_dataset.py` to pick the input file from and write the output file `fun_labels_v<version_id>_combined0_9.csv` in the `data/v<version_id>/csvs/combined0_9` folder
    - Change paths in `code/pipeline/get_cluster_barycenters.py` to pick the input files from and write the output files in the `data/v<version_id>/csvs/combined0_9` folder
  - This produces the final barycenter dataset `barycenter<iteration_number + 1>_v<version_id>_combined0_9.csv` for the given `<version_id>`

## Analysis of Computational Pipeline Output
### Map Network Structures to Functional Clusters
In our analysis, we stopped the computational pipeline after two iterations (i.e., iteration_number = 1). The last barycenter dataset created is `barycenter2_v<version_id>_combined0_9.csv`
   - Run `code/analysis/map_structures_to_func_clusters.py`
   - Input file `fun_labels_v<version_id>_combined0_9.csv`
   - This will create files `final_func_cluster<bary_id>_model_params.csv` with bary_id given by the label in `fun_labels_v<version_id>_combined0_9.csv`
   - The output files will be created in the `data/v<version_id>/csvs/combined0_9/final_func_model_param_map` folder
     
**Get Text IDs of Barycenters**
For each version \
   - Plot the barycenters from the last run of the computational pipeline using the function `plot_conc_vs_time_in_dataset_one_by_one` in `code/analysis/visualize.py`
   - Create a file named `text_id_desc.csv` with two columns: 'text_id' and 'desc'. The 'text_id' column should be populated with a four-letter string for each function and the 'desc' column should have a description of the function based on inspection of the plots. The order of the 'text_id' in this file should follow the bary_id
   - Put this file in the `data/v<version_id>/csvs/combined0_9` folder
  
**Get Functional Cluster Sizes & Mapping of Function IDs with Barycenter IDs**
   - Run `get_fcluster_sizes_bary_id_text_id_maps_combined0_9_datasets.py` with nfunc = number of functional clusters (or number of barycenters) in the last iteration
   - This will create a file `func_id_bary_id_text_id.csv` with the sizes of each functional cluster identified by the corresponding bary_id and text_id. The function_id is assigned according to the descending order of the number of circuits in the functional cluster, i.e., the function exhibited by the highest number of circuits is assigned function_id '01', the second highest as '02', and so on
   - Furthermore, this program will create a file `parameter_count_per_model_fcluster<bary_id>.csv` with the count of the number of parameters for which each network exhibits a given function
   - The output files are created at `data/v<version_id>/csvs/combined0_9`


### Integrate Results Across Versions
   1. Run `get_pairwise_dtw_distances` for pairs of versions passed in version_id1 and version_id2 to find barycenter matches across versions. This calculates the pairwise Dynamic Time Warping (DTW) distance between the barycenter datasets. The file with pairwise DTW distances will be created in `data/integrated_results_v0_v1_v2/csvs/pairwise_barycenter_distances` \
   2. Run `compare_barycenters_datasets.py` to plot heatmaps for pairwise DTW distances among the barycenters of pairs of versions. Take the union of barycenters based on the DTW distances, i.e., for small DTW distances, the barycenter pairs can be considered identical, else they are considered unique\
**NOTE** The text_id for identical barycenters across versions identified in the above step should be identical in the `text_id_desc.csv` file in each of the `data/v<version_id>/csvs/combined0_9` folders for the three version_id
   3. Run `code/analysis/get_integrated_results_from_versions.py` to execute the following functions in sequence:\
      i.   Function `get_union_of_functions_from_different_versions` takes the union of the text_id across versions. This creates the file `text_id_desc.csv` in the `data/integrated_results_v0_v1_v2/csvs` folder\
      ii.  Function `get_fcluster_sizes_bary_id_text_id_maps_after_merge_across_versions` gives the mapping of the integrated functional cluster function_id, text_id and the sizes of functional clusters after merging of functional clusters over the three versions in the file `data/integrated_results_v0_v1_v2/csvs/fcluster_sizes.csv`\
      iii. Function `get_overall_barycenter` calculates the barycenter of the matches found in the three versions. This constitutes the final barycenter dataset obtained from the integration of the results from all three versions. The resulting dataset is created in the file `data/integrated_results_v0_v1_v2/csvs/overall_barycenters_v0_and_v1_and_v2.csv`\
      iv. Function `get_final_model_params_after_merge_across_versions` gives the mapping of functional clusters to the structures in the files `/data/integrated_results_v0_v1_v2/csvs/final_func_model_param_map/final_func_cluster<function_id>_model_params.csv` for each function_id\
      v. Function `get_netk_distribution_over_func_cluster_upset_plots` gives the distribution of network functions when a network exhibits multiple functions. This function creates files `data/integrated_results_v0_v1_v2/csvs/distribution_of_network/fc_model_distribution_<number_of_functions>function.csv` where number_of_functions ranges from 1-20. When we inspect these files, we find that no network exhibits one function and the maximum number of functions any network exhibits is 17. Furthermore, this function generates the Upset plots showing the combinations of functions exhibited by the multifunctional networks

**Summary of results across the 10 partitions of networks and the three versions**
![Consistency of results](https://github.com/user-attachments/assets/25d71102-174d-44b6-a9b8-e76d8cafc7a2)

## UNified FramewOrk for reguLatory Dynamics (UNFOLD)
![insights](https://github.com/user-attachments/assets/d6ce70e3-afab-4a6e-a89a-f6a412d833ee)
![insights_legend_white_background](https://github.com/user-attachments/assets/7d3e6984-b657-427a-bda6-566abb252409)

### Analyse Robustness and Plasticity (Structural Diversity (SD) = 0 plane)
To analyse robustness and plasticity, we consider circuit pairs C<sub>i</sub> and C<sub>j</sub> for which Structural Diversity (SD<sub>ij</sub>) = 0, i.e., they share the same network structure. Hence we calculate the Functional Diversity (FD<sub>ij</sub>) and Parametric Diversity (PD<sub>ij</sub>) for pairwise circuits with a given network structure.\

For this, we run `code/analysis/get_functional_diversity.py`. This code runs the following functions:\
i. Function `get_one_hot_function_codes` encodes each circuit function with a one-hot code of length 20 characters. The output file is `data/integrated_results_v0_v1_v2/csvs/robustness_evolvability_plasticity_canalisation_analysis/function_codes.csv`\
ii. Function `get_k_hot_function_codes` encodes each network with a k-hot code of length 20 characters. The output file is `data/integrated_results_v0_v1_v2/csvs/robustness_evolvability_plasticity_canalisation_analysis/k_hot_function_codes.csv`\
iii. Function `get_pairwise_network_hd_k_hot_codes` finds the Hamming Distance between the k-hot function codes for pairs of networks. The output file is `data/integrated_results_v0_v1_v2/csvs/robustness_evolvability_plasticity_canalisation_analysis/pairwise_network_hd_k_hot_codes.csv`\
iv. Function `get_circuitwise_function_category` finds the category code for each circuit function. The output file is `data/integrated_results_v0_v1_v2/csvs/robustness_evolvability_plasticity_canalisation_analysis/circuitwise_function_category_codes.csv`
v. Function `get_networkpairs_with_zero_k_hot_function_hd` filters all those netwotk pairs that share the same set of functions, i.e., they have HD<sub>k-hot</sub> = 0. The output file is `data/integrated_results_v0_v1_v2/csvs/robustness_evolvability_plasticity_canalisation_analysis/pairwise_sd_pd_zero_fd/frequency_of_zero_k_hot_hd.csv`

The Functional Diversity (FD<sub>ij</sub>) for a pair of circuits is given by the expression:\
  ![image](https://github.com/user-attachments/assets/8017fe04-d674-4260-ad5a-e4b196184e96)
  

   
The Parametric Diversity (PD<sub>ij</sub>) for a pair of circuits is given by the Euclidean Distance of the 21-dimensional parameter sets associated with the circuits.

To optimise the computations, the data files containing function category codes and function codes of all circuits are split into files that contain only the list of circuits that share the same network. For this run function `split_files_networkwise_for_zero_sd` in `code/analysis/split_datafiles.py`. The output files are created in the folder `data/integrated_results_v0_v1_v2/csvs/robustness_evolvability_plasticity_canalisation_analysis/pairwise_pd_fd_zero_sd/networkwise_circuits`. This step ensures that when we take pairs of circuits from each resulting file, the SD<sub>ij</sub>  = 0. 

Now, run function `get_fd_pd_networkwise` in `code/analysis/analyze_robustness_plasticity.py` with the input 'overall_mean_pd' = 0. This will create a file in the folder `data/integrated_results_v0_v1_v2/csvs/robustness_evolvability_plasticity_canalisation_analysis/pairwise_pd_fd_zero_sd` with the mean, count and other statistical details of the Parametric Diversities for each network. From this .csv file, calculate the overall mean Parametric Diversity by dividing the product of elements from the columns named 'count' and 'mean' and dividing the sum of these products by the sum of the column 'count'. Then run `get_fd_pd_networkwise` once again with the input 'overall_mean_pd' now set to the value found from the .csv file. The resulting file in `data/integrated_results_v0_v1_v2/csvs/robustness_evolvability_plasticity_canalisation_analysis/pairwise_pd_fd_zero_sd` will contain the number of robust and plastic circuit pairs having the Functional Diversities '0.0', '0.5' and '1.0' for each network structure.

### Analyse Canalisation
To analyse canalisation, we consider circuit pairs C<sub>i</sub> and C<sub>j</sub> for which Functional Diversity (FD<sub>ij</sub>) = 0, i.e., they have network structures that produce the same set of functions under different parametric conditions (HD<sub>k-hot</sub> = 0) AND either they have the same function (HD<sub>one-hot</sub> = 0) OR they exhibit two functions that belong to the same category (w<sub>ij</sub> = 0). 

To calculate the Structural Diversities (SD<sub>ij</sub>) for all circuit pairs, we only need to find the SD<sub>ij</sub> for all possible network pairs. For this run `code/analysis/get_structural_diversity.py`. This code creates the file `data/common/hamming_distance_all_networks.csv` which has the Hamming Distance between every pair of the 16038 possible three-node networks.

Run `code/analysis/analyze_canalization.py` which will execute the following functions:
i. Function `split_networkpairs_by_sd` to split the network pairs by their Hamming Distance, which is same as their Structural Diversity\
ii. Function `get_number_of_canalised_circuits_by_fid_hd` to get the number of circuits for a pair of functionally canalised (with the function given by the Functional Cluster ID) network pair having Structural Diversities 0-9\
iii. Function `get_number_of_canalised_circuits_by_fcat_hd` to get the number of circuits in a function category that have been canalised\
iv. Function `plot_freq_of_canalised_circuits_vs_sd_for_fcat` and `plot_freq_of_canalised_circuits_vs_sd_overall` to plot the above results


Cite this repository:
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.15387089.svg)](https://doi.org/10.5281/zenodo.15387089)


















