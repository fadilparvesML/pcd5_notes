Repeated code/function:

1. 1044 -> 1166 in date_helper_functions.py 
2. `cluster_include`, `cluster_exclude`, `process_2_txn` and `exception` using the same methods
3. `process_roar_money_account`, `process_none_roar_money_account` and `process_anti_payroll_keyword_present` for the `PD` tagging use the same conditions
4. `weekend` section in `process_none_roar_money_account` and `process_anti_payroll_keyword_present` condition is basically the same
5. 


Ideas:

1. Some date related function that calculates proportion can be removed or joined with the non proportion method and return the output at once.
2. 1044 -> 1166 in date_helper_functions.py can use Template method design pattern to create generic method and just change the validator value in parameter
3. Refactoring long methods into small subset of methods
4. For `process_cluster_include` and `process_cluster_exclude` we can use Facade as the methods are similar with some minor differences, so for each step in the process we allow the subclasses for the `include` and `exclude` to perform it own unique process
5. Use numpy for heavy mathematical calculations 
6. 

Use Flyweight/Cache mechanism to decrease RAM usage and latency
1.  

