Continue after [[Classification]]

1. Call the [[New Flow Classification Combine#classification_segment_checker | classification_segment_checker]] function
2. Create an empty DataFrame `all_income_groups` for final result
3. Loop over each group of `income_id`
4. Get those that are still null payfrequencies or irregular payfrequencies within each income_id that has a detected paycheck and assign those to `null_tagger_txns`
5. Get those that are not null payfrequencies or irregular payfrequencies within each income_id that has a detected paycheck and assign those to `regular_tagger_txns`
6. If length of `regular_tagger_txns` and length or `null_tagger_txns` > 0 then perform the below
	1. Create an empty DataFrame `all_modified_txns`
	2. Loop over each group of `bv_account_id` and `primary_cluster_id` for `null_tagger_txns`
		1. If the length of transactions for the null group is <= 2
			1. Get the number of unique dates in the null group transaction
			2. If the number of unique dates >= 2
				1. Loop over regular_group of `regular_tagger_txns` on `bv_account_id`, `primary_cluster_id` and `tagger`
					1. Call the [[New Flow Classification Combine#process_2_cluster |process2cluster]] function that takes in the regular_group and null_group
					2. If the return is True then run the [[New Flow Classification Combine#tag_classification_txns |tag_classification_txns]] function with the `null_group` and `regular_group` as input and the output is stored in `modified_null_txns` 
					3. Else `modified_null_txns` is copy of the `null_group` and `tagger` is set as `PD| irregular`
			3. Else
				1. Loop over regular_group of `regular_tagger_txns` on `bv_account_id`, `primary_cluster_id` and `tagger`
				2. Call the [[New Flow Classification Combine#process_1_cluster |process_1_cluster]] function with `regular_group` and `null_group` as input
				3. If the return is True then run the [[New Flow Classification Combine#tag_classification_txns |tag_classification_txns]] function with `regular_group` and `null_group` as input and the output is stored in `modified_null_txns`
				4. Else `modified_null_txns` is a copy of `null_group` and `tagger` is set as `PD|single`
		2. Else
			1. Set `modified_null_txns` as copy of `null_tagger_txns`
			2. Set the `tagger` as `PD| irregular`
		3. Append the `modified_null_txns` into `all_modified_txns`
	3. Create `modified_group` df that is basically concatenation of `all_modified_txns` and `regular_tagger_txns`
7. Else 
	1. `modifeid_group` is just copy of the `group`
8. Append all the `modified_group` to `all_income_groups` 
9. If `tagger` is in all_income_groups then update the `pcd_txn_type` column based on if the `tagger` is null or not, set it to `paycheck` or `potential_paycheck`
10. Return the `all_income_groups` dataframe.

# classification_segment_checker

1. Create empty dataframe named all_income_group
2. Create groups of income id
3. For each cluster (income_id) and its group then create child group on `primary_cluster_id`
4. For each primary cluster and its group get the first `tagger` and `income_type`
5. If the frequency is `irregular` and income type is `others` call [[New Flow Classification Combine#clustering_segmentor | clustering_segmentor]] function and then call the [[Classification#process_cluster_include]]
6. Else set new_txns as the group copy
7. Append the new_txns to all_income_groups dataframe
8. Return the all_income_groups dataframe

# clustering_segmentor

1. An empty DataFrame `modified_txns` is created to store the final results.
2. The `unique_cluster_id` is split on the underscore character (`_`), and the first part is stored in `prefix`.
3. The function then loops over each unique value in the `unique_cluster_id` column of the `transactions` DataFrame.
4. For each unique cluster ID, the function extracts the corresponding transactions from the `transactions` DataFrame into `clustered_transactions`.
5. If the number of transactions in `clustered_transactions` is greater than 4, the function proceeds with the following steps:
6. The transactions in `clustered_transactions` are sorted by the `transaction_date` column in ascending order.
	
	1. The time difference between successive transactions is calculated and stored in the `time_diff` column.
	2. A rolling window is applied to the `time_diff` column, and the mode (or mean if no mode exists) of each window is calculated and stored in the `rolling_mode` column.
	3. The `rolling_mode` values are shifted up by one row, and the last row is filled with the previous row's value. This is stored in the `rolling_mode_shiftup` column.
	4. A new group is created whenever there is a difference between the `rolling_mode` and `rolling_mode_shiftup` columns. The cumulative sum of these differences is stored in the `new_group` column.
	5. The number of transactions for each new group is calculated and stored in the `num_txns` column.
	6. If the number of transactions for a new group is less than 3, the group is ignored (set to NaN). Otherwise, the group is kept. This is stored in the `new_group_fix` column.
	7. The `new_group_fix` values are forward-filled and backward-filled to handle NaN values, and the result is stored in the `new_group_final` column.
	8. The function then loops over each group in `new_group_final`, adds a cluster ID to the group if the group index is greater than 0, and appends the group to `all_groups`.
	9. The function drops several intermediate columns from `all_groups`.
	
6. If the number of transactions in `clustered_transactions` is not greater than 4, `all_groups` is simply set to a copy of `clustered_transactions`.
7. Whether `all_groups` was created from the detailed process or simply copied from `clustered_transactions`, it is appended to `modified_txns`.
8. After looping over all unique cluster IDs, the function returns `modified_txns`, which contains the segmented transactions.

# process_2_cluster

for matched descriptions, where clusters with at least 2 transaction but doesn't have a payfrequency or is irregular/single is given a chance to check if they can take the same payfrequency as clusters with regular payfrequency, with slightly looser rules.

1. The function retrieves the last non-null value from the `tagger` column of the `notnull_cluster_transactions` DataFrame and stores it in the `cluster_tagger` variable.
2. The `null_cluster_transactions` DataFrame is sorted in ascending order by the `transaction_date` column.
3. An empty list `diff` is initialized with `np.nan` as its first element.
4. The function then loops over each transaction in `null_cluster_transactions` using a for loop. The loop index `i` ranges from 0 to one less than the number of transactions.
5. Inside the loop, the function checks if the current transaction is not the last one. If it's not, it further checks if the date of the next transaction (`null_cluster_transactions["transaction_date"].iloc[i + 1]`) is not equal to the date of the current transaction (`null_cluster_transactions["transaction_date"].iloc[i]`).
6. If the dates are not equal, it calculates the difference in days between the next transaction date and the current transaction date. This difference is stored in the `delta` variable and then appended to the `diff` list.
7. If the dates are equal, the function checks if the current transaction is the first one. If it is, it calculates the difference in days between the next transaction date and the current transaction date, just like before. This difference is then appended to the `diff` list.
8. If the current transaction is not the first one, the function appends the most recent non-zero difference in days (stored in `delta`) to the `diff` list.
9. After the loop, the function converts the `diff` list to a pandas Series and assigns it to the `se` variable.
10. The function then adds a new column `time_diff` to the `null_cluster_transactions` DataFrame, which contains the values from the `se` Series.
11. The function creates a temporary DataFrame `null_cluster_transactions_temp` by dropping rows from `null_cluster_transactions` that have NaN values in the `time_diff` column.
12. If the number of transactions in `null_cluster_transactions_temp` is greater than 2, the function calls the `remove_quantile_outlier` function to remove the top and bottom 20% of time differences. The resulting DataFrame replaces `null_cluster_transactions_temp`
13. Compute mean and std of remaining time differences
14. If cluster tagger is `weekly` or `PD|weekly` and then the mean is > 6 and < 16 then return True
15. If cluster tagger is `semi-monthly`, `bi-weekly` or `PD|bi_semi` then the mean > 12 and < 16 then return True
16. If cluster tagger in `monthly` or `PD|monthly` and then the mean > 27 and < 32 then return true
17. Else return True
18. Return False if no condition above from step 14 to 17 meets

# process_1_cluster

1. The function retrieves the last value from the `tagger` column of the [`notnull_cluster_transactions`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") DataFrame and stores it in the [`cluster_tagger`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") variable.
2. The function creates a list [`expanded_timestamplist`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") that contains the dates of transactions in [`notnull_cluster_transactions`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") plus and minus one day.
3. The function converts the list of TimeStamp objects in [`expanded_timestamplist`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") to a list of strings for comparison.
4. The function checks if the date of the transaction in [`null_cluster_transactions`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") matches any date in [`expanded_timestamplist`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py"). If there is a match, the function returns `True`.
5. If there is no match, the function calculates the difference in days between the date of the transaction in [`null_cluster_transactions`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") and the date of the last transaction in [`notnull_cluster_transactions`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py"). This difference is stored in the [`delta`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") variable.
6. The function then checks the value of [`cluster_tagger`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py"):
    1. If [`cluster_tagger`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") is 'weekly' or 'PD|weekly', the function checks if [`delta`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") is between 6 and 8. If it is, the function returns `True`.
    2. If [`cluster_tagger`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") is 'semi-monthly', 'bi-weekly', or 'PD|bi_semi', the function checks if [`delta`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") is between 12 and 16. If it is, the function returns `True`.
    3. If [`cluster_tagger`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") is 'monthly' or 'PD|monthly', the function checks if [`delta`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") is between 27 and 32. If it is, the function returns `True`.
    4. If [`cluster_tagger`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") is none of the above, the function returns `True`.
7. If none of the above conditions are met, the function returns `False`.

# tag_classification_txns

1. The function defines two lists [`target_columns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") and [`target_columns_2`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") that contain the names of the columns to be used later.
2. An empty DataFrame [`tagged_txns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") is created to store the final results.
3. The function then loops over each unique combination of `primary_cluster_id` and `income_id` in the [`changed_txns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") DataFrame.
4. For each unique combination, the function extracts the corresponding transactions from the [`changed_txns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") DataFrame into [`group`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py").
5. The function retrieves the first row of [`txns2`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") DataFrame and stores the values of the columns specified in [`target_columns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") into [`cluster_info`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py").
6. The function also stores the values of the columns specified in [`target_columns_2`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") from the first row of [`txns2`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") DataFrame into [`cluster_info_2`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py").
7. The function creates a new DataFrame [`tagged_required_txns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") by appending [`group`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") DataFrame to it. It also assigns the values in [`cluster_info`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") to the corresponding columns in [`tagged_required_txns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py").
8. The function then loops over each column in [`target_columns_2`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py"), and assigns the corresponding value in [`cluster_info_2`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") to all rows of that column in [`tagged_required_txns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py").
9. The function appends [`tagged_required_txns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") to [`tagged_txns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py").
10. The function checks if the `pattern` column in [`tagged_txns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py") contains certain values and if the `account_type` column is 'roarmoney'. If both conditions are met, it appends '_adjusted' to the `pattern` column. Otherwise, it keeps the `pattern` column as is.
11. After looping over all unique combinations of `primary_cluster_id` and `income_id`, the function returns [`tagged_txns`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/classification/classification_helpers.py"), which contains the tagged transactions.

# Function calls

| Name                           | Parameters                                                              | Returns                 | Number of times called | Required data/feature                                                                                                                                                                                                                                                                                                                | Parent                                                             |
| ------------------------------ | ----------------------------------------------------------------------- | ----------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| classification_segment_checker | processed_transactions -> df                                            | all_income_groups -> df | 1                      | `income_id`<br>`primary_cluster_id`<br>`tagger`<br>`income_type`                                                                                                                                                                                                                                                                     | [[New Flow Classification Combine]]                                |
| clustering_segmentor           | transactions -> df<br><br>unique_cluster_id -> str<br><br>window -> int | modified_txns -> df     | 1                      | `unique_income_id`<br>`transaction_date`<br>`bv_transaction_id`<br>                                                                                                                                                                                                                                                                  | [[New Flow Classification Combine#classification_segment_checker]] |
| process_2_cluster              | regular_txn -> df<br><br>null_txn -> df                                 | bool                    | 1                      | `tagger`<br>`transaction_date`                                                                                                                                                                                                                                                                                                       | [[New Flow Classification Combine]]                                |
| process_1_cluster              | regular_txn -> df<br><br>null_txn -> df                                 | bool                    | 1                      | `tagger`<br>`transaction_date`                                                                                                                                                                                                                                                                                                       | [[New Flow Classification Combine]]                                |
| tag_classification_txns        | regular_txn -> df<br><br>null_txn -> df                                 | tagged_txns -> df       | 1                      | `pattern`<br>`mean`<br>`std`<br>`lower_bound`<br>`upper_bound`<br>`tagger`<br>`pattern`<br>`pcd_txn_type`<br>`wom_pairs`<br>`dom_paydates_semimonthly`<br>`dom_paydates_monthly`<br>`nth_bday_mode_beginning_of_month`<br>`nth_bday_mode_middle_of_month`<br>`nth_bday_mode_end_of_month`<br>`primary_cluster_id`<br>`income_id`<br> | [[New Flow Classification Combine]]                                |
|                                |                                                                         |                         |                        |                                                                                                                                                                                                                                                                                                                                      |                                                                    |
