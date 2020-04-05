# model_build_package

This is a git containing pl/sql code that generates model build code.  Super simple to use and very flexible.


```
CREATE OR REPLACE PROCEDURE create_config_table IS 
BEGIN
-- admin needs to: grant create table to ml_user

begin
execute immediate 'DROP table model_build_settings';
exception when others then null;
end;

execute immediate 'create table model_build_settings (setting_name varchar2(30),setting_value varchar2(4000))';

execute immediate 'insert into model_build_settings values (''ALGO_NAME'', ''ALGO_DECISION_TREE'')';
execute immediate 'insert into model_build_settings values (''PREP_AUTO'', ''ON'')';
execute immediate 'insert into model_build_settings values (''ODMS_TEXT_POLICY_NAME'',''MY_POLICY'')';
execute immediate 'insert into model_build_settings values (''TREE_TERM_MAX_DEPTH'', 7)';
execute immediate 'insert into model_build_settings values (''TREE_TERM_MINREC_SPLIT'', 20)';
execute immediate 'insert into model_build_settings values (''TREE_TERM_MINPCT_SPLIT'', .1)';
execute immediate 'insert into model_build_settings values (''TREE_TERM_MINREC_NODE'', 10)';
execute immediate 'insert into model_build_settings values (''TREE_TERM_MINPCT_NODE'', 0.05)';
commit;

begin
execute immediate 'DROP table model_config';
exception when others then null;
end;

execute immediate 'create table model_config ( '||
'model_name				varchar2(100) '||
', apply_table_name			varchar2(100) '||
', lift_table_name			varchar2(100) '||
', algorithm_name			varchar2(100) '||
', train_table_Name			varchar2(100) '||
', build_settings_table_name	varchar2(100) '||
', case_id				varchar2(100) '||
', target_column_name		VARCHAR2(100) '||
', mining_function			VARCHAR2(100) '||
', test_table_name			VARCHAR2(100) '||
', positive_target_value		VARCHAR2(100) '||
', confusion_matrix_table_name VARCHAR2(100))';

execute immediate 'insert into model_config values(''ALL_CASES_DT'',''ALL_CASES_APPLY_RESULT_DT'',''ALL_CASES_LIFT_DT'',''ALGO_DECISION_TREE'',''ALL_CASES_TRAIN_T'',''MODEL_BUILD_SETTINGS'',''INCIDENT_NUMBER'',''CATEGORIZATION_TIER_2'',''CLASSIFICATION'',''ALL_CASES'',''SW'',''ALL_CASES_CONFUSION_MATRIX_DT'')';
execute immediate 'insert into model_config values(''ALL_CASES_SVM'',''ALL_CASES_APPLY_RESULT_SVM'',''ALL_CASES_LIFT_SVM'',''ALGO_SUPPORT_VECTOR_MACHINES'',''ALL_CASES_TRAIN_T'',''MODEL_BUILD_SETTINGS'',''INCIDENT_NUMBER'',''CATEGORIZATION_TIER_2'',''CLASSIFICATION'',''ALL_CASES'',''SW'',''ALL_CASES_CONFUSION_MATRIX_SVM'')';
execute immediate 'insert into model_config values(''ALL_CASES_RF'',''ALL_CASES_APPLY_RESULT_RF'',''ALL_CASES_LIFT_RF'',''ALGO_RANDOM_FOREST'',''ALL_CASES_TRAIN_T'',''MODEL_BUILD_SETTINGS'',''INCIDENT_NUMBER'',''CATEGORIZATION_TIER_2'',''CLASSIFICATION'',''ALL_CASES'',''SW'',''ALL_CASES_CONFUSION_MATRIX_RF'')';
execute immediate 'insert into model_config values(''ALL_CASES_NN'',''ALL_CASES_APPLY_RESULT_NN'',''ALL_CASES_LIFT_NN'',''ALGO_NEURAL_NETWORK'',''ALL_CASES_TRAIN_T'',''MODEL_BUILD_SETTINGS'',''INCIDENT_NUMBER'',''CATEGORIZATION_TIER_2'',''CLASSIFICATION'',''ALL_CASES'',''SW'',''ALL_CASES_CONFUSION_MATRIX_NN'')';
execute immediate 'insert into model_config values(''ALL_CASES_NB'',''ALL_CASES_APPLY_RESULT_NB'',''ALL_CASES_LIFT_NB'',''ALGO_NAIVE_BAYES'',''ALL_CASES_TRAIN_T'',''MODEL_BUILD_SETTINGS'',''INCIDENT_NUMBER'',''CATEGORIZATION_TIER_2'',''CLASSIFICATION'',''ALL_CASES'',''SW'',''ALL_CASES_CONFUSION_MATRIX_NB'')';
commit;

END create_config_table;
/

---------------------------------------------------------
CREATE OR REPLACE PROCEDURE build_models IS 
v_accuracy number;
-- admin needs to:
-- grant create mining model to ml_user;
-- grant execute on dbms_data_mining to ml_user;
BEGIN

begin
execute immediate 'DROP TABLE all_lift_data_cases PURGE';
exception when others then null;
end;

begin
execute immediate 'CREATE TABLE all_lift_data_cases (algo_name VARCHAR2(50), QUANTILE_NUMBER NUMBER, GAIN_CUMULATIVE NUMBER)';
exception when others then null;
end;

-- loop through algorithms

FOR i IN (select * from model_config) LOOP

execute immediate 'delete from '||i.build_settings_table_name||' where setting_name = ''ALGO_NAME''';
execute immediate 'insert into '||i.build_settings_table_name||' select ''ALGO_NAME'', '''||i.algorithm_name||''' from dual';

begin
dbms_data_mining.drop_model(i.model_name);
exception when others then null;
end;

begin
dbms_data_mining.create_model(
	model_name		=> i.model_name,
	mining_function	=> i.mining_function, 
	data_table_name	=> i.train_table_name, 
	case_id_column_name => i.case_id,
	target_column_name	=> i.target_column_name, 
	settings_table_name	=> i.build_settings_table_name);
end;

-- drop apply result

begin
execute immediate 'drop table '||i.apply_table_name||' purge';
execute immediate 'drop table '||i.lift_table_name||' purge';
exception when others then null;
end;

-- test the model by generating a apply result and then create a lift result

begin
dbms_data_mining.apply(
	model_name		=> i.model_name,
	data_table_name	=> i.test_table_name,
	case_id_column_name	=> i.case_id,
	result_table_name	=> i.apply_table_name);
exception when others then null;
end;

begin
dbms_data_mining.compute_lift(
	apply_result_table_name		=> i.apply_table_name,
	target_table_name			=> i.test_table_name,
	case_id_column_name			=> i.case_id,
	target_column_name			=> i.target_column_name,
	lift_table_name			=> i.lift_table_name,
	positive_target_value		=> i.positive_target_value,
	score_column_name			=> 'PREDICTION',
	score_criterion_column_name => 'PROBABILITY',
	num_quantiles				=> 100);
exception when others then null;
end;

begin
execute immediate 'insert into all_lift_data_cases select '''||i.algorithm_name||''', QUANTILE_NUMBER, GAIN_CUMULATIVE from '||i.lift_table_name;
exception when others then null;
end;

-- build confusion matrix table

begin
EXECUTE IMMEDIATE 'drop table '||i.confusion_matrix_table_name;
exception when others then null;
end;

begin
DBMS_DATA_MINING.COMPUTE_CONFUSION_MATRIX (
	accuracy                     => v_accuracy,
	apply_result_table_name      => i.apply_table_name,
	target_table_name            => i.test_table_name,
	case_id_column_name          => i.case_id,
	target_column_name           => i.target_column_name,
	confusion_matrix_table_name  => i.confusion_matrix_table_name,
	score_column_name            => 'PREDICTION',
	score_criterion_column_name  => 'PROBABILITY',
	score_criterion_type         => 'PROBABILITY');
DBMS_OUTPUT.PUT_LINE(i.model_name||' Accuracy: ' || ROUND(v_accuracy * 100,2));
end;

END LOOP;

END build_models;
/
```