# Oracle PL/SQL Packages to Automate Oracle Machine Learning Model Builds

There are several good examples and tutorials/workshops that show how to write sql and pl/sql that create machine learning models.  To simplify the process this package was created as a stored procedure, that when invoked creates and populates a settings table, a configuration table, and then provides a second procedure to build the machine learning artifacts (models, lift tables, apply tables, and confusion matrix tables).  The process is as follows:

## **Step 1:** Grant user privileges

- Normally dm_role (or oml_developer) roles are sufficient to build models.  However when pl/sql is executed in stored procedures the grants must be direct and not through a role.  Execute the following grants from admin (or sys in dbcs) to ml_user (or other ml user name)
```
<copy>grant unlimited storage to tablespace data;
grant execute on dbms_data_mining to ml_user;
grant create mining model to ml_user;
grant create table to ml_user;</copy>
```

## **Step 2:** Compile the package.

- Copy/paste the code block below into sqlplus or sqldeveloper.

![](./images/001.png  " ")


## **Step 3:** Execute the create_config procedure.

- Execute the create_config procedure.
```
<copy>exec model_build_pkg.create_config;</copy>
```

![](./images/002.png  " ")

- This creates and populates the MODEL_BUILD_SETTINGS table (normally used in ML builds) and a MODEL_CONFIG table.  **This procedure only needs to be run once.  If it is re-run it drops and creates and re-populates the tables.  You must review the contents and adjust to your particular case.  The existing values are only used as a sample.

![](./images/003.png  " ")

- The MODEL_BUILD_SETTINGS table has the following values.  Note in this particular case a text index was used and therefore has text index settings - these would not normally be required and you can remove those rows.

![](./images/004.png  " ")

- The MODEL_CONFIG table has the following values.  The model build procedure (run next) loops through this table and builds a model for each row (in this case five models and related tables are built.  You must update this table with your own values.

![](./images/005.png  " ")

## **Step 4:** Execute the build_models procedure.

- If you have not first run the create_config procedure you will be notified as such.
```
<copy>set serveroutput on
exec model_build_pkg.build_models;</copy>
```

![](./images/006.png  " ")

- Now that you have created and updated the configuration you can build the models.  This may take some time, depending on the number of models and the data volume.
```
<copy>exec model_build_pkg.build_models;</copy>
```


