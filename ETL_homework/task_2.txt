
/* ******************************************************************************************************************************
 * Scenario 2: Multivalued Relationships. Part 1
 * 
 * Inputs:
 *   4 target tables: 
 *     1) #dim_technician (already prepolulated)
 *     2) #dim_technician_group: table where each combination of people (each group) would have just one record 
 *     3) #bridge_technician_group: bridge table which would bound #dim_technician with #dim_technician_group
 *     4) #fact_inspection with granularity of one record per inspection.
 * 
 *   1 input table:
 *     1) staging fact-like table with inspections info. Each inspection(defined by inspection_id) is performed by a couple of employees
 *     
 ****************************************************************************************************************************** */

------------------------------------------------------------------------------------------------------------------
-- Initialization (target tables)
------------------------------------------------------------------------------------------------------------------
drop table if exists #dim_technician;
create table #dim_technician(
    employee_sk int not null identity primary key,
    employee_bk varchar(255),
    full_name  varchar(255)
);
truncate table #dim_technician;
insert into #dim_technician(employee_bk, full_name) 
values 
('uk100', 'Lori Smith'),
('uk101', 'Ivan Kohut'),
('uk102', 'Oksana Lysytsia'),
('uk103', 'Iryna Vovk'),
('uk104', 'Mag Smith'),
('uk105', 'Ivanka Kohut'),
('uk106', 'Leyla Lysytsia'),
('uk107', 'Olesia Vovk'),
('uk108', 'John Smith'),
('uk109', 'Ivan Piven'),
('uk110', 'Oksana Baran'),
('uk111', 'Iryna Zayets');

drop table if exists #dim_technician_group;
create table #dim_technician_group(
    employee_group_sk int not null identity primary key,
    group_code varchar(1024)
);

drop table if exists #bridge_technician_group;
create table #bridge_technician_group(
    employee_group_sk int,
    employee_sk int
);

drop table if exists #fact_inspection;
create table #fact_inspection(
    inspection_id int not null primary key,
    employee_group_sk int
);


------------------------------------------------------------------------------------------------------------------
-- Initialization (source data)
------------------------------------------------------------------------------------------------------------------
drop table if exists #stg_inspection;
create table #stg_inspection(
  inspection_id int not null,
  employee_bk varchar(255) not null,
  primary key (inspection_id, employee_bk)
);  
insert into #stg_inspection values
(1,'uk100'),(1,'uk101'),(2,'uk100'),(2,'uk101'),(3,'uk100'),(3,'uk101'),(3,'uk102'),(4,'uk102'),
(4,'uk103'),(4,'uk104'),(5,'uk102'),(5,'uk103'),(6,'uk105'),(6,'uk106'),(6,'uk107'),(7,'uk107'),
(7,'uk105'),(7,'uk106'),(8,'uk106'),(8,'uk107'),(8,'uk105'),(9,'uk109'),(10,'uk109'),(10,'uk110'),
(11,'uk109'),(11,'uk110'),(11,'uk111'),(12,'uk108'),(12,'uk109'),(13,'uk107'),(13,'uk108'),
(14,'uk107'),(14,'uk108'),(15,'uk108'),(15,'uk107'),(16,'uk104'),(16,'uk105'),(16,'uk106'),
(17,'uk104'),(17,'uk105'),(17,'uk106'),(18,'uk104'),(18,'uk105'),(18,'uk106'),
(19,'uk103'),(19,'uk104'),(19,'uk105');

/* ******************************************************************************************************************************
 * Exersize #2 (initial population):
 * Prepare a SQL script (sequence of SQL statements) to populate the first time all the empty target tables:
 *   1) #dim_technician_group;
 *   2) #bridge_technician_group;
 *   3) #fact_inspection;
 ****************************************************************************************************************************** */

/*   1) #dim_technician_group; 
* combine all employees which belong to the group */

SELECT 
    inspection_id,
    STRING_AGG(employee_bk, ',') WITHIN GROUP (ORDER BY employee_bk) AS group_code
INTO #stg_groups
FROM #stg_inspection
GROUP BY inspection_id

SELECT * 
FROM #stg_groups

/* create record if not exists */
INSERT INTO #dim_technician_group (group_code)
SELECT DISTINCT group_code FROM #stg_groups
WHERE NOT EXISTS (SELECT 1 FROM #dim_technician_group d WHERE d.group_code = #stg_groups.group_code)

SELECT * FROM #dim_technician_group;

/* 3) #fact_inspection */
INSERT into #fact_inspection (inspection_id, employee_group_sk)
SELECT sg.inspection_id, dtg.employee_group_sk
FROM #dim_technician_group dtg
INNER JOIN #stg_groups sg 
ON dtg.group_code = sg.group_code 

SELECT * from #fact_inspection

/* bridge */
INSERT INTO #bridge_technician_group
SELECT DISTINCT 
    dtg.employee_group_sk,
    dt.employee_sk
FROM #dim_technician_group dtg
CROSS APPLY STRING_SPLIT(dtg.group_code, ',') s
INNER JOIN #dim_technician dt 
    ON dt.employee_bk = s.value


SELECT * FROM #bridge_technician_group

/*
SELECT 
    dt.employee_bk,
    dt.full_name,
    COUNT(fi.inspection_id) AS total_inspections
FROM #dim_technician dt
INNER JOIN #bridge_technician_group btg 
    ON dt.employee_sk = btg.employee_sk
INNER JOIN #fact_inspection fi 
    ON btg.employee_group_sk = fi.employee_group_sk
GROUP BY dt.employee_bk, dt.full_name
ORDER BY total_inspections DESC, dt.employee_bk;
*/


/* ******************************************************************************************************************************
 * Scenario 3: Multivalued Relationships. Part 2
 * 
 * Incremental data processing. After the initial data population the new portion of the source data has been arrived.
 * The requirements are to incrementally merge this data into the target tables.
 *     
 ****************************************************************************************************************************** */

/* ******************************************************************************************************************************
-- New portion of the source data
****************************************************************************************************************************** */
insert into #stg_inspection values
(21,'uk102'),(21,'uk103'),
(22,'uk107'),(22,'uk106'),(22,'uk105'),
(23,'uk103'),(23,'uk104'),(23,'uk105'),
(24,'uk100'),(24,'uk105'),
(25,'uk100'),(25,'uk103'),(25,'uk111'),
(26,'uk100'),(26,'uk105'),
(27,'uk100'),(27,'uk103'),(27,'uk111');

/* ******************************************************************************************************************************
 * Exersize #3 (incremental processing):
 * Prepare a SQL script (sequence of SQL statements) to merge the new portion of the data into the target table:
 *   1) #dim_technician_group;
 *   2) #bridge_technician_group;
 *   3) #fact_inspection;
 ****************************************************************************************************************************** */
DROP TABLE IF EXISTS #stg_groups_new;

SELECT 
    inspection_id,
    STRING_AGG(employee_bk, ',') WITHIN GROUP (ORDER BY employee_bk) AS group_code
INTO #stg_groups_new
FROM #stg_inspection
GROUP BY inspection_id;

SELECT * FROM #stg_groups_new;

/* insert new team, check that it does not exists in technician group */

INSERT INTO #dim_technician_group (group_code)
SELECT DISTINCT sg.group_code
FROM #stg_groups_new sg
WHERE NOT EXISTS (
    SELECT 1 
    FROM #dim_technician_group dtg 
    WHERE dtg.group_code = sg.group_code
);

SELECT * FROM #dim_technician_group;

/* insert new combination if it does not exist */

INSERT INTO #bridge_technician_group (employee_group_sk, employee_sk)
SELECT DISTINCT 
    dtg.employee_group_sk,
    dt.employee_sk
FROM #dim_technician_group dtg
CROSS APPLY STRING_SPLIT(dtg.group_code, ',') s
INNER JOIN #dim_technician dt 
    ON dt.employee_bk = s.value
WHERE NOT EXISTS (
    SELECT 1 
    FROM #bridge_technician_group btg 
    WHERE btg.employee_group_sk = dtg.employee_group_sk 
      AND btg.employee_sk = dt.employee_sk
);

SELECT * FROM #bridge_technician_group;

INSERT INTO #fact_inspection (inspection_id, employee_group_sk)
SELECT sg.inspection_id, dtg.employee_group_sk
FROM #dim_technician_group dtg
INNER JOIN #stg_groups_new sg 
    ON dtg.group_code = sg.group_code 
WHERE NOT EXISTS (
    SELECT 1 
    FROM #fact_inspection fi 
    WHERE fi.inspection_id = sg.inspection_id
);

SELECT * FROM #fact_inspection;