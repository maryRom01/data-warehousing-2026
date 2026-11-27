
/* ******************************************************************************************************************************
 * Scenario 4: Build a special bridge table for the organization structure (using Closure table approach)
 * 
 * Inputs:
 *   2 target tables: 
 *     1) #dim_employee (already prepolulated)
 *     2) #bridge_employee_hierachy table where hierarchical data should be populate according to Closure table approach  
 * 
 *   1 input table:
 *     1) #stg_employee table where each record contains parent_id
 *     
 ****************************************************************************************************************************** */

------------------------------------------------------------------------------------------------------------------
-- Initialization (target tables)
------------------------------------------------------------------------------------------------------------------
drop table if exists #dim_employee;
create table #dim_employee(
    employee_sk int not null identity primary key,
    employee_bk varchar(255),
    full_name  varchar(255)
);
truncate table #dim_employee;
insert into #dim_employee(employee_bk, full_name) 
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

drop table if exists #bridge_employee_hierachy;
create table #bridge_employee_hierachy(
    ancestor_employee_sk int not null,
    descendant_employee_sk int not null,
    depth_from_parent int,
    is_leaf bit,
    primary key (ancestor_employee_sk,descendant_employee_sk)
);


------------------------------------------------------------------------------------------------------------------
-- Initialization (target tables)
------------------------------------------------------------------------------------------------------------------
drop table if exists #stg_employee;
create table #stg_employee(
    employee_bk varchar(255) not null primary key,
    full_name  varchar(255),
    parent_bk varchar(255)
);

truncate table #stg_employee;
insert into #stg_employee(employee_bk, full_name, parent_bk) 
values 
('uk100', 'Lori Smith',null),
('uk101', 'Ivan Kohut','uk100'),
('uk102', 'Oksana Lysytsia','uk100'),
('uk103', 'Iryna Vovk','uk101'),
('uk104', 'Mag Smith','uk101'),
('uk105', 'Ivanka Kohut','uk102'),
('uk106', 'Leyla Lysytsia','uk102'),
('uk107', 'Olesia Vovk','uk104'),
('uk108', 'John Smith','uk104'),
('uk109', 'Ivan Piven','uk107'),
('uk110', 'Oksana Baran','uk108'),
('uk111', 'Iryna Zayets','uk109');


/* ******************************************************************************************************************************
 * Exersize #4 (build hierarchy bridge table):
 * Prepare a SQL script (sequence of SQL statements) to build the #bridge_employee_hierachy table according to closure table algorithm
 ****************************************************************************************************************************** */
/*
'uk100'
    'uk101'
        'uk103'
        'uk104'
            'uk107'
                'uk109'
                    'uk111'
            'uk108'
                'uk110'
    'uk102'
        'uk105'
        'uk106'
*/

WITH hierarchy_cte AS (
    -- pair employee with themselves at depth 0
    SELECT 
        employee_bk AS ancestor_bk,
        employee_bk AS descendant_bk,
        0 AS depth_from_parent
    FROM #stg_employee

    UNION ALL

    -- (recursive)
    SELECT 
        cte.ancestor_bk,
        emp.employee_bk AS descendant_bk,
        cte.depth_from_parent + 1
    FROM hierarchy_cte cte
    INNER JOIN #stg_employee emp
        ON cte.descendant_bk = emp.parent_bk
)

INSERT INTO #bridge_employee_hierachy (
    ancestor_employee_sk, 
    descendant_employee_sk, 
    depth_from_parent, 
    is_leaf
)
SELECT 
    anc.employee_sk AS ancestor_employee_sk,
    dsc.employee_sk AS descendant_employee_sk,
    cte.depth_from_parent,
    CASE 
        WHEN EXISTS (
            SELECT 1 
            FROM #stg_employee child 
            WHERE child.parent_bk = cte.descendant_bk
        ) THEN 0 
        ELSE 1 
    END AS is_leaf
FROM hierarchy_cte cte
INNER JOIN #dim_employee anc 
    ON cte.ancestor_bk = anc.employee_bk
INNER JOIN #dim_employee dsc 
    ON cte.descendant_bk = dsc.employee_bk;

SELECT * 
FROM #bridge_employee_hierachy
ORDER BY ancestor_employee_sk, depth_from_parent, descendant_employee_sk;

