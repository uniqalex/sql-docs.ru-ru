---
title: COALESCE (Transact-SQL) | Документы Майкрософт
ms.custom: ''
ms.date: 08/30/2017
ms.prod: sql
ms.prod_service: database-engine, sql-database, sql-data-warehouse, pdw
ms.component: t-sql|language-elements
ms.reviewer: ''
ms.suite: sql
ms.technology: t-sql
ms.tgt_pltfrm: ''
ms.topic: language-reference
f1_keywords:
- COALESCE
- COALESCE_TSQL
dev_langs:
- TSQL
helpviewer_keywords:
- expressions [SQL Server], nonnull
- COALESCE function
- first nonnull expressions [SQL Server]
- nonnull expressions
ms.assetid: fafc0dba-f8a8-4aad-9b7f-908e34b74d88
caps.latest.revision: 52
author: douglaslMS
ms.author: douglasl
manager: craigg
monikerRange: '>= aps-pdw-2016 || = azuresqldb-current || = azure-sqldw-latest || >= sql-server-2016 || = sqlallproducts-allversions'
ms.openlocfilehash: cce959876ded6e2815260e82b0c9e909e804fd5a
ms.sourcegitcommit: 1740f3090b168c0e809611a7aa6fd514075616bf
ms.translationtype: HT
ms.contentlocale: ru-RU
ms.lasthandoff: 05/03/2018
---
# <a name="coalesce-transact-sql"></a>COALESCE (Transact-SQL)
[!INCLUDE[tsql-appliesto-ss2008-all-md](../../includes/tsql-appliesto-ss2008-all-md.md)]

Вычисляет аргументы по порядку и возвращает текущее значение первого выражения, изначально не вычисленного как `NULL`. Например, `SELECT COALESCE(NULL, NULL, 'third_value', 'fourth_value');` возвращает третье значение, так как это первое значение, не равное NULL. 
  
 ![Значок ссылки на раздел](../../database-engine/configure-windows/media/topic-link.gif "Значок ссылки на раздел") [Синтаксические обозначения в Transact-SQL](../../t-sql/language-elements/transact-sql-syntax-conventions-transact-sql.md)  
  
## <a name="syntax"></a>Синтаксис  
  
```  
COALESCE ( expression [ ,...n ] )   
```  
  
## <a name="arguments"></a>Аргументы  
 *expression*  
 [Выражение](../../t-sql/language-elements/expressions-transact-sql.md) любого типа данных.  
  
## <a name="return-types"></a>Типы возвращаемых данных  
 Возвращает тип данных аргумента *expression* с наиболее высоким приоритетом. Если ни одно из выражений не допускает значения NULL, то результат типизируется как не допускающий значения NULL.  
  
## <a name="remarks"></a>Remarks  
 Если все аргументы имеют значение `NULL`, `COALESCE` возвращает `NULL`. По крайней мере одно из значений NULL должно быть типизированным `NULL`.  
  
## <a name="comparing-coalesce-and-case"></a>Сравнение COALESCE и CASE  
 Выражение `COALESCE` — синтаксический ярлык для выражения `CASE`.  Это означает, что код `COALESCE`(*expression1*,*...n*) переписывается оптимизатором запросов как следующее выражение `CASE`:  
  
 ```sql  
 CASE  
 WHEN (expression1 IS NOT NULL) THEN expression1  
 WHEN (expression2 IS NOT NULL) THEN expression2  
 ...  
 ELSE expressionN  
 END  
 ```  
  
 Это означает, что входные значения (*expression1*, *expression2*, *expressionN* и т. д.) вычисляются многократно. Кроме того, в соответствии со стандартом SQL выражение значения, содержащее вложенный запрос, считается недетерминированным и вложенный запрос вычисляется дважды. В любом случае могут быть возвращены различные результаты для первого и последующих вычислений.  
  
 Например, если выполняется код `COALESCE((subquery), 1)`, вложенный запрос вычисляется дважды. В результате можно получить различные результаты в зависимости от уровня изоляции запроса. Например, код может вернуть `NULL` при уровне изоляции `READ COMMITTED` в многопользовательской среде. Чтобы обеспечить устойчивые результаты, используйте уровень изоляции `SNAPSHOT ISOLATION` или замените `COALESCE` функцией `ISNULL`. Кроме того, можно переписать запрос, чтобы поместить вложенный запрос в подзапрос выборки, как показано в следующем примере:  
  
```sql  
SELECT CASE WHEN x IS NOT NULL THEN x ELSE 1 END  
from  
(  
SELECT (SELECT Nullable FROM Demo WHERE SomeCol = 1) AS x  
) AS T;  
  
```  
  
## <a name="comparing-coalesce-and-isnull"></a>Сравнение COALESCE и ISNULL  
 Функция `ISNULL` и выражение `COALESCE` имеют аналогичные цели, но могут отличаться поведением.  
  
1.  Так как `ISNULL` — это функция, она вычисляется только один раз.  Как было сказано выше, входные значения для выражения `COALESCE` могут вычисляться несколько раз.  
  
2.  Различается определение типа данных результирующего выражения. `ISNULL` использует тип данных первого параметра, `COALESCE` следует правилам выражения `CASE` и возвращает тип данных значения, первым удовлетворившим условие `WHEN`.
  
3.  Для `ISNULL` и `COALESCE` различается допустимость значения NULL для результирующего выражения. Возвращаемое значение `ISNULL` всегда считается не допускающим значения NULL (предполагается, что возвращаемое значение не допускает значения NULL), в то время как `COALESCE` с параметрами, отличными от NULL, считается имеющим значение `NULL`. Таким образом, выражения `ISNULL(NULL, 1)` и `COALESCE(NULL, 1)`, несмотря на одинаковый синтаксис, имеют разные значения допустимости значений NULL. Это проявляется при использовании данных выражений в вычисляемых столбцах, создании ограничений ключа или детерминировании возвращаемого значения определяемой пользователем скалярной функции для возможности индексации, как показано в приведенном ниже примере.  
  
    ```sql  
    USE tempdb;  
    GO  
    -- This statement fails because the PRIMARY KEY cannot accept NULL values  
    -- and the nullability of the COALESCE expression for col2   
    -- evaluates to NULL.  
    CREATE TABLE #Demo   
    (   
    col1 integer NULL,   
    col2 AS COALESCE(col1, 0) PRIMARY KEY,   
    col3 AS ISNULL(col1, 0)   
    );   
  
    -- This statement succeeds because the nullability of the   
    -- ISNULL function evaluates AS NOT NULL.  
  
    CREATE TABLE #Demo   
    (   
    col1 integer NULL,   
    col2 AS COALESCE(col1, 0),   
    col3 AS ISNULL(col1, 0) PRIMARY KEY   
    );  
    ```  
  
4.  Проверки для `ISNULL` и `COALESCE` также различаются. Например, значение `NULL` для `ISNULL` преобразуется в значение **int**, а для `COALESCE` необходимо предоставить тип данных.  
  
5.  Функция `ISNULL` принимает только два параметра, тогда как `COALESCE` принимает переменное количество параметров.  
  
## <a name="examples"></a>Примеры  
  
### <a name="a-running-a-simple-example"></a>A. Выполнение простого примера  
 В следующем примере показано, как функция `COALESCE` выбирает из первого столбца данные, отличные от значения NULL. В этом примере используется база данных [!INCLUDE[ssSampleDBnormal](../../includes/sssampledbnormal-md.md)].  
  
```sql  
SELECT Name, Class, Color, ProductNumber,  
COALESCE(Class, Color, ProductNumber) AS FirstNotNull  
FROM Production.Product;  
```  
  
### <a name="b-running-a-complex-example"></a>Б. Выполнение сложного примера  
 В следующем примере таблица `wages` включает три столбца с данными о ежегодной заработной плате сотрудников: hourly_wage, salary и commission. Однако служащий получает только один тип выплат. Для определения общей оплаты для всех служащих используйте функцию `COALESCE` для получения не равных NULL значений столбцов `hourly_wage`, `salary` и `commission`.  
  
```sql  
SET NOCOUNT ON;  
GO  
USE tempdb;  
IF OBJECT_ID('dbo.wages') IS NOT NULL  
    DROP TABLE wages;  
GO  
CREATE TABLE dbo.wages  
(  
    emp_id        tinyint   identity,  
    hourly_wage   decimal   NULL,  
    salary        decimal   NULL,  
    commission    decimal   NULL,  
    num_sales     tinyint   NULL  
);  
GO  
INSERT dbo.wages (hourly_wage, salary, commission, num_sales)  
VALUES  
    (10.00, NULL, NULL, NULL),  
    (20.00, NULL, NULL, NULL),  
    (30.00, NULL, NULL, NULL),  
    (40.00, NULL, NULL, NULL),  
    (NULL, 10000.00, NULL, NULL),  
    (NULL, 20000.00, NULL, NULL),  
    (NULL, 30000.00, NULL, NULL),  
    (NULL, 40000.00, NULL, NULL),  
    (NULL, NULL, 15000, 3),  
    (NULL, NULL, 25000, 2),  
    (NULL, NULL, 20000, 6),  
    (NULL, NULL, 14000, 4);  
GO  
SET NOCOUNT OFF;  
GO  
SELECT CAST(COALESCE(hourly_wage * 40 * 52,   
   salary,   
   commission * num_sales) AS money) AS 'Total Salary'   
FROM dbo.wages  
ORDER BY 'Total Salary';  
GO  
```  
  
 [!INCLUDE[ssResult](../../includes/ssresult-md.md)]  
  
 ```   
 Total Salary  
 ------------  
 10000.00  
 20000.00  
 20800.00  
 30000.00  
 40000.00  
 41600.00  
 45000.00  
 50000.00  
 56000.00  
 62400.00  
 83200.00  
 120000.00  
  
 (12 row(s) affected)
 ```  
  
### <a name="c-simple-example"></a>В. Простой пример  
 В приведенном ниже примере показано, как `COALESCE` выбирает данные из первого столбца, содержащего значение, отличное от NULL. В этом примере предполагается, что таблица `Products` содержит следующие данные:  
  
 ```  
 Name         Color      ProductNumber  
 ------------ ---------- -------------  
 Socks, Mens  NULL       PN1278  
 Socks, Mens  Blue       PN1965  
 NULL         White      PN9876
 ```  
  
 Затем выполняется следующий запрос COALESCE:  
  
```sql  
SELECT Name, Color, ProductNumber, COALESCE(Color, ProductNumber) AS FirstNotNull   
FROM Products ;  
```  
  
 [!INCLUDE[ssResult](../../includes/ssresult-md.md)]  
  
 ```  
 Name         Color      ProductNumber  FirstNotNull  
 ------------ ---------- -------------  ------------  
 Socks, Mens  NULL       PN1278         PN1278  
 Socks, Mens  Blue       PN1965         Blue  
 NULL         White      PN9876         White
 ```  
  
 Обратите внимание на то, что в первой строке значение `FirstNotNull` равно `PN1278`, а не `Socks, Mens`. Это связано с тем, что в примере столбец `Name` не был указан в качестве параметра для `COALESCE`.  
  
### <a name="d-complex-example"></a>Г. Сложный пример  
 В приведенном ниже примере `COALESCE` используется для сравнения значений в трех столбцах и возврата только значения, отличного от NULL, найденного в столбцах.  
  
```sql  
CREATE TABLE dbo.wages  
(  
    emp_id        tinyint   NULL,  
    hourly_wage   decimal   NULL,  
    salary        decimal   NULL,  
    commission    decimal   NULL,  
    num_sales     tinyint   NULL  
);  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (1, 10.00, NULL, NULL, NULL);  
  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (2, 20.00, NULL, NULL, NULL);  
  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (3, 30.00, NULL, NULL, NULL);  
  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (4, 40.00, NULL, NULL, NULL);  
  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (5, NULL, 10000.00, NULL, NULL);  
  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (6, NULL, 20000.00, NULL, NULL);  
  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (7, NULL, 30000.00, NULL, NULL);  
  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (8, NULL, 40000.00, NULL, NULL);  
  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (9, NULL, NULL, 15000, 3);  
  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (10,NULL, NULL, 25000, 2);  
  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (11, NULL, NULL, 20000, 6);  
  
INSERT INTO dbo.wages (emp_id, hourly_wage, salary, commission, num_sales)  
VALUES (12, NULL, NULL, 14000, 4);  
  
SELECT CAST(COALESCE(hourly_wage * 40 * 52,   
   salary,   
   commission * num_sales) AS decimal(10,2)) AS TotalSalary   
FROM dbo.wages  
ORDER BY TotalSalary;  
```  
  
 [!INCLUDE[ssResult](../../includes/ssresult-md.md)]  
  
 ```
 Total Salary  
 ------------  
 10000.00  
 20000.00  
 20800.00  
 30000.00  
 40000.00  
 41600.00  
 45000.00  
 50000.00  
 56000.00  
 62400.00  
 83200.00  
 120000.00
 ```  
  
## <a name="see-also"></a>См. также:  
 [Функция ISNULL (Transact-SQL)](../../t-sql/functions/isnull-transact-sql.md)   
 [CASE (Transact-SQL)](../../t-sql/language-elements/case-transact-sql.md)  
  
  
