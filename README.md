Úvod
Tento projekt se zaměřuje na analýzu dostupnosti základních potravin v České republice a porovnání s dalšími evropskými státy. Naším cílem je vytvořit robustní datový podklad, který umožní odpovědět na klíčové výzkumné otázky týkající se vývoje mezd, cen potravin a vlivu makroekonomických ukazatelů (např. HDP a GINI koeficient) na životní úroveň občanů.

Cíle projektu
Analýza mezd a cen: Prozkoumat, zda mzdy rostou ve všech odvětvích, nebo se v některých případech snižují.
Dostupnost potravin: Vyhodnotit, kolik litrů mléka a kilogramů chleba je možné zakoupit v prvním a posledním srovnatelném období dostupných dat.
Rychlost zdražování: Identifikovat potravinové kategorie, u kterých je meziroční nárůst cen nejnižší.
Ekonomický vliv: Zjistit, zda výraznější růst HDP v daném roce koreluje s rychlejším růstem mezd a cen potravin.
Použité datové sady
Primární tabulky:

czechia_payroll – Data o mzdách v různých odvětvích za několik let.
czechia_payroll_calculation, czechia_payroll_industry_branch, czechia_payroll_unit, czechia_payroll_value_type – Číselníky související s mzdovými daty.
czechia_price – Data o cenách vybraných potravin.
czechia_price_category – Číselník kategorií potravin.
Sdílené a dodatečné tabulky:

czechia_region a czechia_district – Číselníky regionů a okresů České republiky.
countries – Informace o zemích (např. hlavní město, měna, národní jídlo).
economies – Makroekonomické ukazatele jako HDP, GINI koeficient a další.
Výstupy projektu
Výsledkem práce jsou dvě hlavní tabulky, ze kterých lze získat data potřebná pro odpovědi na výzkumné otázky:

Primární tabulka: t_{jmeno}_{prijmeni}_project_SQL_primary_final – Konsolidovaná data o mzdách a cenách potravin pro Českou republiku (sjednocená na srovnatelná období).
Sekundární tabulka: t_{jmeno}_{prijmeni}_project_SQL_secondary_final – Dodatečná data o HDP, GINI a populaci pro další evropské státy.
Kromě těchto tabulek obsahuje repozitář také sadu SQL skriptů, které generují potřebné datové přehledy a umožňují odpovědět na definované výzkumné otázky. Tyto otázky mohou výsledky podporovat nebo vyvracet – záleží na tom, co ukazují analyzovaná data.

Postup:
Vytvoření tabulky
CREATE TABLE t_Rudolf_Salficky_project_SQL_primary_final AS 
WITH avg_salaries AS (
    -- Calculate YoY percentage change in food prices
     SELECT 
        rl.payroll_year, 
        rl.industry_branch_code,
        ib.name AS industry_name,
        AVG(rl.value) AS avg_salary
    FROM czechia_payroll AS rl
     RIGHT JOIN czechia_payroll_industry_branch AS ib
        ON rl.industry_branch_code = ib.code
    WHERE rl.value_type_code = 5958 
      AND rl.value IS NOT NULL 
      AND rl.industry_branch_code IS NOT NULL
    GROUP BY payroll_year, industry_branch_code
),
avg_prices AS (SELECT 
        EXTRACT(YEAR FROM cp.date_from) AS year,  
        cpc.name AS product_name,  
        SUM(cp.value * (cp.date_to - cp.date_from)) 
            / CASE 
                WHEN SUM(cp.date_to - cp.date_from) = 0 THEN 1  -- Avoid division by zero
                ELSE SUM(cp.date_to - cp.date_from) 
              END AS avg_price  -- Weighted average
    FROM czechia_price AS cp
    RIGHT JOIN czechia_price_category AS cpc
        ON cp.category_code = cpc.code
    WHERE cpc.name IS NOT NULL  -- Consider all food categories
    GROUP BY year, cpc.name)
SELECT payroll_year, industry_branch_code,industry_name, avg_salary, NULL AS product, NULL AS price FROM avg_salaries
UNION ALL
SELECT year, NULL AS salary, NULL AS industry, NULL AS industry_name, product_name, avg_price FROM avg_prices;


Druhá tabulka: 
CREATE TABLE t_Rudolf_Salficky_project_SQL_secondary_final AS 
SELECT e.country, e.year, e.GDP, e.population, e.gini, e.fertility, e.mortaliy_under5
FROM economies e
INNER JOIN countries c ON e.country = c.country;

--------------------------------------------------------------------------------------------------------------------------------------
First question: Rostou v průběhu let mzdy ve všech odvětvích nebo v některých klesají?
WITH min_max_years AS (
    -- Get min and max payroll year for each industry
    SELECT 
        industry_branch_code, 
        MIN(payroll_year) AS min_year, 
        MAX(payroll_year) AS max_year
    FROM t_Rudolf_Salficky_project_SQL_primary_final
    GROUP BY industry_branch_code
),
salary_changes AS (
    -- Join back to get avg salaries for min and max years
    SELECT 
        m.industry_branch_code,
        m.min_year, 
        y1.avg_salary AS min_year_salary,
        m.max_year, 
        y2.avg_salary AS max_year_salary,
        y2.avg_salary - y1.avg_salary AS salary_difference
    FROM min_max_years m
    JOIN t_Rudolf_Salficky_project_SQL_primary_final y1 
        ON m.industry_branch_code = y1.industry_branch_code 
       AND m.min_year = y1.payroll_year
    JOIN t_Rudolf_Salficky_project_SQL_primary_final y2 
        ON m.industry_branch_code = y2.industry_branch_code 
       AND m.max_year = y2.payroll_year
)
SELECT 
    SUM(CASE WHEN salary_difference < 0 THEN 1 ELSE 0 END) AS decreasing_count,
    SUM(CASE WHEN salary_difference > 0 THEN 1 ELSE 0 END) AS increasing_count,
    CASE 
        WHEN SUM(CASE WHEN salary_difference < 0 THEN 1 ELSE 0 END) = 0 
        THEN 'Mzdy rostou ve všech odvětvích'
        ELSE 'V některých odvětvích mzdy klesají'
    END AS result
FROM salary_changes; 

Odpověď: 
0	19	Mzdy rostou ve všech odvětvích

--------------------------------------------------------------------------------------------------------------------------------------
Second question: Kolik je možné si koupit litrů mléka a kilogramů chleba za první a poslední srovnatelné období?

WITH price_data AS (
    -- Get average prices for mléko and chléb for the first and last years
    SELECT 
        payroll_year, 
        product, 
        price 
    FROM t_Rudolf_Salficky_project_SQL_primary_final
    WHERE product LIKE '%mléko%' OR product LIKE '%chléb%'
),
salary_data AS (
    -- Get average salary for the first and last years
    SELECT 
        payroll_year, 
        AVG(avg_salary) AS avg_salary
    FROM t_Rudolf_Salficky_project_SQL_primary_final
    WHERE avg_salary IS NOT NULL
    GROUP BY payroll_year
),
first_last_years AS (
    -- Get the first and last year that exists in both datasets
    SELECT 
        MIN(price_data.payroll_year) AS first_year, 
        MAX(price_data.payroll_year) AS last_year
    FROM price_data
    INNER JOIN salary_data 
        ON price_data.payroll_year = salary_data.payroll_year
)
SELECT 
    p.payroll_year,
    p.product,
    s.avg_salary,
    p.price,
    s.avg_salary / p.price AS purchasable_quantity
FROM price_data p
JOIN salary_data s ON p.payroll_year = s.payroll_year
JOIN first_last_years f ON p.payroll_year IN (f.first_year, f.last_year)
ORDER BY p.payroll_year, p.product;

Odpověď:
2006	Chléb konzumní kmínový 	1281kilogramů
2006	Mléko polotučné pasterované	1438 litrů
2018	Chléb konzumní kmínový	1342 kilogramů
2018	Mléko polotučné pasterované	1641 litrů

--------------------------------------------------------------------------------------------------------------------------------------
Third question: Která kategorie potravin zdražuje nejpomaleji (nejnižší meziroční procentuální nárůst)?

WITH price_changes AS (
    -- Calculate year-over-year percentage change
    SELECT 
        p1.payroll_year, 
        p1.product, 
        p1.price AS current_price, 
        p2.price AS previous_price,
        CASE 
            WHEN p2.price = 0 OR p2.price IS NULL THEN NULL  -- Avoid division by zero
            ELSE ((p1.price - p2.price) / p2.price) * 100
        END AS percent_increase
    FROM t_Rudolf_Salficky_project_SQL_primary_final p1
    LEFT JOIN t_Rudolf_Salficky_project_SQL_primary_final p2
        ON p1.product = p2.product
        AND p1.payroll_year = p2.payroll_year + 1  -- Join with the previous year's data
)
-- Find the category with the lowest average yearly growth rate
SELECT 
    product, 
    AVG(percent_increase) AS avg_annual_increase
FROM price_changes
WHERE percent_increase IS NOT NULL  
GROUP BY product
ORDER BY avg_annual_increase ASC
LIMIT 1;  -- Get the category with the slowest price increase

Odpověď:
Cukr krystalový	-1.95010693394638

--------------------------------------------------------------------------------------------------------------------------------------
Fourth question: Existuje rok, ve kterém byl meziroční nárůst cen potravin výrazně vyšší než růst mezd (rozdíl větší než 10 %)?

Pokud mají potraviny být odděleně
WITH price_growth AS (
    -- Calculate YoY percentage change in food prices
    SELECT 
        p1.payroll_year,
        p1.product,
        p1.price,
        p1.price - p0.price AS absolute_change,
        (p1.price - p0.price) / p0.price * 100 AS price_growth
    FROM t_Rudolf_Salficky_project_SQL_primary_final p1
    JOIN t_Rudolf_Salficky_project_SQL_primary_final p0 ON p1.payroll_year = p0.payroll_year + 1 AND p1.product = p0.product
),
salary_growth AS (
    -- Calculate YoY percentage change in salaries
    SELECT 
        s1.payroll_year AS year,
        AVG(s1.avg_salary) AS avg_salary,
        AVG(s1.avg_salary) - AVG(s0.avg_salary) AS absolute_change,
        (AVG(s1.avg_salary) - AVG(s0.avg_salary)) / AVG(s0.avg_salary) * 100 AS salary_growth
    FROM t_Rudolf_Salficky_project_SQL_primary_final s1
    JOIN t_Rudolf_Salficky_project_SQL_primary_final s0 ON s1.payroll_year = s0.payroll_year + 1
    GROUP BY s1.payroll_year
),
combined_growth AS (
    -- Join price and salary growth data for years that exist in both datasets
    SELECT 
        pg.payroll_year, 
        pg.product, 
        pg.price_growth, 
        sg.salary_growth, 
        pg.price_growth - sg.salary_growth AS difference
    FROM price_growth pg
    JOIN salary_growth sg ON pg.payroll_year = sg.year
)-- Find years where price growth exceeded salary growth by more than 10%
SELECT * 
FROM combined_growth
WHERE difference > 10
ORDER BY payroll_year, product;

Odpověď: 39 potravin

Pokud mají potraviny za rok být dohromady – tak takový rok není
WITH price_growth AS (
    -- Calculate YoY percentage change in food prices for each product
    SELECT 
        p1.payroll_year,
        p1.price,
        p1.price - p0.price AS absolute_change,
        (p1.price - p0.price) / p0.price * 100 AS price_growth
    FROM t_Rudolf_Salficky_project_SQL_primary_final p1
    JOIN t_Rudolf_Salficky_project_SQL_primary_final p0 ON p1.payroll_year = p0.payroll_year + 1 AND p1.product = p0.product
),
avg_price_growth AS (
    -- Aggregate the price growth per year by averaging across all products
    SELECT 
        payroll_year, 
        AVG(price_growth) AS avg_price_growth
    FROM price_growth
    GROUP BY payroll_year
),
salary_growth AS (
    -- Calculate YoY percentage change in salaries
    SELECT 
        s1.payroll_year AS year,
        AVG(s1.avg_salary) AS avg_salary,
        AVG(s1.avg_salary) - AVG(s0.avg_salary) AS absolute_change,
        (AVG(s1.avg_salary) - AVG(s0.avg_salary)) / AVG(s0.avg_salary) * 100 AS salary_growth
    FROM t_Rudolf_Salficky_project_SQL_primary_final s1
    JOIN t_Rudolf_Salficky_project_SQL_primary_final s0 ON s1.payroll_year = s0.payroll_year + 1
    GROUP BY s1.payroll_year
),
combined_growth AS (
    -- Join average price growth and salary growth for matching years
    SELECT 
        pg.payroll_year, 
        pg.avg_price_growth, 
        sg.salary_growth, 
        pg.avg_price_growth - sg.salary_growth AS difference
    FROM avg_price_growth pg
    JOIN salary_growth sg ON pg.payroll_year = sg.year
)
-- Find years where price growth exceeded salary growth by more than 10%
SELECT * 
FROM combined_growth
WHERE difference > 10
ORDER BY payroll_year;

Fifth question: Má výška HDP vliv na změny ve mzdách a cenách potravin?

WITH price_growth AS (
    -- Calculate YoY percentage change in food prices for each product
    SELECT 
        p1.payroll_year,
        p1.price,
        p1.price - p0.price AS absolute_change,
        (p1.price - p0.price) / p0.price * 100 AS price_growth
    FROM t_Rudolf_Salficky_project_SQL_primary_final p1
    JOIN t_Rudolf_Salficky_project_SQL_primary_final p0 ON p1.payroll_year = p0.payroll_year + 1 AND p1.product = p0.product
),
avg_price_growth AS (
    -- Aggregate the price growth per year by averaging across all products
    SELECT 
        payroll_year, 
        AVG(price_growth) AS avg_price_growth
    FROM price_growth
    GROUP BY payroll_year
),
salary_growth AS (
    -- Calculate YoY percentage change in salaries
    SELECT 
        s1.payroll_year AS year,
        AVG(s1.avg_salary) AS avg_salary,
        AVG(s1.avg_salary) - AVG(s0.avg_salary) AS absolute_change,
        (AVG(s1.avg_salary) - AVG(s0.avg_salary)) / AVG(s0.avg_salary) * 100 AS salary_growth
    FROM t_Rudolf_Salficky_project_SQL_primary_final s1
    JOIN t_Rudolf_Salficky_project_SQL_primary_final s0 ON s1.payroll_year = s0.payroll_year + 1
    GROUP BY s1.payroll_year
),
gdp_growth AS (
    -- Calculate YoY percentage change in GDP
    SELECT 
        s1.year,
        AVG(s1.GDP),
        AVG(s1.GDP) - AVG(s0.GDP) AS absolute_GDP,
        (AVG(s1.GDP) - AVG(s0.GDP)) / AVG(s0.GDP) * 100 AS GDP_growth
    FROM t_Rudolf_Salficky_project_SQL_secondary_final s1
    JOIN t_Rudolf_Salficky_project_SQL_secondary_final s0 ON s1.year = s0.year + 1
    GROUP BY s1.year
),
combined_growth AS (
    -- Join average price growth, GDP growth and salary growth for matching years
    SELECT 
        pg.payroll_year, 
        pg.avg_price_growth, 
        sg.salary_growth,
        gg.gdp_growth
    FROM avg_price_growth pg
    JOIN salary_growth sg ON pg.payroll_year = sg.YEAR
    JOIN gdp_growth gg ON pg.payroll_year = gg.YEAR
)
-- Find years where price growth exceeded salary growth by more than 10%
SELECT * 
FROM combined_growth
ORDER BY payroll_year;

Odpověď: Pokles GDP zpětně ovlivňuje ceny potravin a mzdy
