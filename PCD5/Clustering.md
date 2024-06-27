Continue after [[Preprocessing]]

NEWFLOW
1. Calls the `new_flow_clustering` method, which performs the below:
	1. Get all the unique `income_types` from potential paychecks
	2. Create empty dataframe called all_transactions
	3. For each income type call the `clustering_by_income_type` method that takes in `potential_paychecks, client_id, income_type and is_compare_sec_cluster_id` and perform the below:
		1. Create empty transactions and clustered transactions dataframe
		2. Call the `remove_columns_before` method to remove some columns based on the process argument, in this case it is `clustering` 
		3. Filter for transactions based on the current in loop income_type
		4. Call the `primary_clustering` method that performs the below (input: `client_id, transactions`):
			1. Set max dendogram cutoff at 0.01
			2. If there is more than 2 transactions, perform the below:
				1. Create TfidfVectorizer and TfidfVectorizer with percentages instances
				2. Convert all the `feed` from transactions to list
				3. Fit the feeds to the tfidf models and if there is error then fit the feeds to the tfidf with percentage model
				4. Calculate the distance by 1 - cosine_similarity(transactions_matrix)
				5. Define the linkage_matrix using ward clustering pre-computed distances
				6. Perform the clustering using fcluster
				7. Obtain the cluster values and add them to `cluster` column
				8. Add a `cluster_id` by calling the add_cluster_id on each transaction row and passing the `cluster_type` as primary (input: single transaction row and `cluster_type`, output: transactions with cluster_type_cluster_id)
				9. Convert the cluster id as string
			3. If there is less than 2 transactions then perform below:
				1. Add random cluster number
				2. Add unique cluster id
				3. Convert the cluster id as string
			4. Return the clustered transactions
		5. If income type is unemployment benefit then called the `uib_cluster_segmentor` method which performs the below: (input: `primary_cluster_id, transaction_date`, output: `time_diff, time_diff_outliers, new_group_final`)
			1. Create empty `modified_txns` dataframe
			2. Get the prefix from the unique cluster id
			3. For each cluster id in the list of unique cluster ids, do the below:
				1. Filter for transactions which the given primary cluster id
				2. If the length of the filtered transactions is more than 2 then run below:
					1. Sort the values by transaction date in ascending order
					2. Create `time_diff` column by subtracting current date with the previous one and only take days
					3. Create `time_diff_outliers` if `time_diff` is more than 35 days and cast it as integer
					4. Create `new_group_final` by taking cumsum of `time_diff_outliers`
					5. Create `all_groups` empty dataframe
					6. For each index and group that is grouped by `new_group_final`, perform the below:
						1. If index is more than 0:
							1. Create copy of group into new group
							2. Add cluster id to the new group
							3. Cast it as string
						2. Else
							1. Create copy of group into new group
						3. All groups append new group
					7. All groups drop the 3 created columns
				3. Else
					1. All groups is just the copy of input transactions
			4. `modified_txns` is result of appending all the all_groups together
			5. Return `modified_txns`
		6. Perform the secondary clustering by calling `secondary_cluster_clustering` function which performs the below:
			1. Sort the transactions date by transaction date ascending
			2. If the length of transactions is more than 2 then perform the below:
				1. Create cluster feed frequency variable by grouping clustered transactions on primary cluster id and feed, counting the `bv_transaction_id`
				2. Create rank on the feed frequency by grouping on primary cluster id column and using the `feed_frequency` to rank
				3. Get the most frequent feed per cluster
				4. Check for clusters that had no frequent feed
				5. check if list `no_freq_feed` is non-empty. If it is, the code filters a DataFrame `transactions` based on whether the values in the column `unique_cluster_id` are in `no_freq_feed`. It then sorts the filtered DataFrame by the `transaction_date`, groups it by `unique_cluster_id`, and selects the last row of each group. The resulting DataFrame, containing only the columns `unique_cluster_id` and "feed", is assigned to `last_feed_cluster`. If `no_freq_feed` is empty, `last_feed_cluster` is assigned an empty DataFrame.
				6. Combine all chosen unique feed per primary cluster together
				7. Take the feed from the chosen unique feed per primary cluster together
				8. If the length of unique cluster transactions is more than or equal to 2, then do the below:
					1. Create instance of tfidf and tfidf with percentage
					2. Fit the tfidf to the feeds and if it fails then fit the tfidf with percentage 
					3. Call the `awesome_cossim_top` function that is optimized cosine similarity function that only stores the top 5 (don't think users will split paychecks more than 5 times) most similar items and only items with a similarity above 0.8 and the matches are used as sparse matrix in `get_matches_df`
					4. Call the `get_matches_df` function that performs the below:
						1. Create non zeros sparse matrix
						2. Get the sparse matrix rows and cols
						3. If top is more than sprascols size then set `nr_matches = top` else `nr_matches = sparsecols.size`
						4. Create empty `left_side and right_side` array of size nr_matches
						5. For each index from 0 to `nr_matches` add the values to left and right side and compute the similarity by calling sparse matrix data
						6. Return dataframe with the left and right side and similarity
					5. Remove all the matches that are exactly the same with itself
					6. If length of `matches_df` is more than 0, the below will happen:
						1. The first part of the code is removing duplicate pairs from [`matches_df`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py"). It does this by combining the values in the "left_side" and "right_side" columns into a list, sorting this list, and then dropping duplicates. Next, the code groups [`matches_df`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") by the "right_side" and "left_side" columns and applies the `list` function to the "similarity", "left_side", and "right_side" columns. This results in new columns "new_similarity_right", "new_left_side", "new_right_side", and "new_similarity_left" that contain lists of values for each group. The "new_left_side" and "new_right_side" columns are then combined into a new column "combined_feed", which is then transformed into a list of unique, sorted values. This column is then converted to a string and stored in "combined_feed_str". The code then groups [`matches_df`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") by "combined_feed_str" and applies the `list` function to the "similarity" column, resulting in a new column "combined_similarity". The [`add_temporary_cluster_id`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") function is then applied to [`matches_df`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") grouped by "combined_feed_str", adding a temporary cluster ID to each group. This ID is then converted to a string. The code then merges [`matches_df`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") with [`unique_cluster_transactions`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") twice, first on "left_side" and then on "right_side". The resulting DataFrame [`second_merge`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") has its "secondary_cluster_id_temp", "secondary_cluster_similarity_score", and "secondary_cluster_feed_list" columns filled with the first non-null values from the corresponding columns in [`matches_df`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py"). This DataFrame is then merged with [`transactions`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py"), and the "secondary_cluster_id_temp" column is filled with the first non-null values from the corresponding column in [`transactions`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") or the [`unique_cluster_id`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") column. The [`add_cluster_id`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") function is then applied to [`transactions`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") grouped by "secondary_cluster_id_temp", adding a final, reproducible cluster ID to each group. This ID is then converted to a string. Finally, the temporary cluster ID column is dropped from [`transactions`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py"), and any duplicate rows based on the "bv_transaction_id" column are dropped. The [`add_cluster_id`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") function sorts a DataFrame by "transaction_date", then generates a cluster ID based on the cluster type, client ID, and feed. The [`add_temporary_cluster_id`](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html "algo_pcd5_src/module/clustering/clustering.py") function simply adds a temporary cluster ID to a DataFrame. Both functions return the modified DataFrame.
					7. Else:
						1. Add `secondary_cluster_id` to transaction with the value being the primary cluster id
						2. Set `secondary_cluster_similarity_score and secondary_cluster_feed_list` as nan
				9. Else
					1. Add `secondary_cluster_id` to transaction with the value being the primary cluster id
					2. Set `secondary_cluster_similarity_score and secondary_cluster_feed_list` as nan
			3. Else
				1. Add `secondary_cluster_id` to transaction with the value being the primary cluster id
				2. Set `secondary_cluster_similarity_score and secondary_cluster_feed_list` as nan
			4. Return the second level clustered transactions
		7. If `is_compare_sec_cluster_id` True, then do the below:
			1. Call the `matching_secondary_clusters_new_flow` function that will perform the below:
				1. Filter for transactions where the secondary cluster id has more than 0 transactions
				2. Get a list of unique feed that corresponds to highest feed for a given sec_cluster_id by calling `compile_highest_rank_or_latest_feed`
				3. Call `group_base_feed_new_flow` to get back all the matches
				4. If length of the matches is more than 0, do the following:
					1. Group by matches with `matched_feed` and get the matched cluster id
					2. Create new transactions variable my merging transactions with matches with left join on secondary cluster id
					3. Add a matched cluster id and drop the matched cluster id temp column
				5. Else
					1. Create new transactions by copying the transactions
					2. Set `feed_dist_count` as nan
					3. Set `matched_cluster_id` as `secondary_cluster_id`
				6. Return the new transactions
			2. Assign the `income_id` as the `matched_clustered_id`
		8. Return the second level clustered transactions
	4. Return the clustered transactions