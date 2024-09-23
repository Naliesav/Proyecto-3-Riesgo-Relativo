### **Procesamiento y análisis:**

1. **Conectar/importar datos a otras herramientas.**
    a. Descargar y descomprimir data set.
    b. Subir archivos a BigQuery en formato CSV.

![Untitled](https://github.com/user-attachments/assets/702ec3a4-edba-4761-a305-dd64cb230b01)


2. **Identificar y manejar valores nulos.**

a) La única tabla que presenta valores nulos es: “user_info”

```sql
SELECT
  *
FROM
  `proyecto-riesgo-relativo-3.prestamos.user_info`
WHERE
  user_id IS NULL
  OR age IS NULL
  OR sex IS NULL
  OR last_month_salary IS NULL
  OR number_dependents IS NULL;
```

Se utilizó la siguiente fórmula y dio un total de 7.199 valores nulos en las columnas “last_month_salary” y “number_dependents” . En este caso se dejarán los datos hasta concretar si se pueden descartar o no.

De igual manera confirmaremos una información. En la tabla “default” está la columna “default_flag” y en la tabla “user_info” está la columna “last_month_salary”, las juntaremos con LEFT JOIN para ver si en mayor proporción son los clientes con bandera 1 (mal pagador).

```sql
SELECT
  d.user_id,
  d.default_flag,
  u.last_month_salary
FROM
  `proyecto-riesgo-relativo-3.prestamos.default` AS d
LEFT JOIN
  `proyecto-riesgo-relativo-3.prestamos.user_info` AS u
ON
  d.user_id = u.user_id;
```

![Untitled (1)](https://github.com/user-attachments/assets/f76cb660-af29-4869-be50-ba005c1e5bd3)


Esta fórmula y tabla abarcan 36.000 usuarios, ahora veremos cuántos de ellos son mal pagadores:

```sql
SELECT
  d.user_id,
  d.default_flag,
  u.last_month_salary
FROM
  `proyecto-riesgo-relativo-3.prestamos.default` AS d
LEFT JOIN
  `proyecto-riesgo-relativo-3.prestamos.user_info` AS u
ON
  d.user_id = u.user_id
WHERE
  d.default_flag = 1;
```

![Untitled (2)](https://github.com/user-attachments/assets/a53b79d7-09da-4167-a50b-cc9a8c82a1e5)

Aquí nos refleja que 683 usuarios son mal pagadores. Tendremos este dato guardado para profundizar más adelante.

3. **Identificar y manejar valores duplicados.**

En las tablas generales no se encuentran datos duplicados a simple vista. Por ejemplo, con la tabla “user_info” se hizo la siguiente consulta:

```sql
SELECT
  user_id,
  age,
  sex,
  last_month_salary,
  number_dependents,
  COUNT(*) AS count
FROM
  `proyecto-riesgo-relativo-3.prestamos.user_info`
GROUP BY
  user_id,
  age,
  sex,
  last_month_salary,
  number_dependents
HAVING
  COUNT(*) > 1
ORDER BY
  count DESC;
```

Y arroja que no hay datos duplicados el líneas generales. Mencionar también que los números de “user_id” son únicos. Si realizamos la fórmula columnas por columnas, ejemplo:

```sql
SELECT
  age,
  COUNT(*) AS count
FROM
  `proyecto-riesgo-relativo-3.prestamos.user_info`
GROUP BY
  age
HAVING
  COUNT(*) > 1
ORDER BY
  count DESC;
```

Nos dirá la cantidad de “duplicados” con respecto a la edad, es decir, nos va arrojar una tabla mostrando cuántas personas hay con ciertas edades:

![Untitled (3)](https://github.com/user-attachments/assets/fcc58ab4-8dd8-4894-9664-c179047a10f1)

A pesar de que no es una información duplicada como tal, nos ayuda a realizar el conteo de cuántas personas hay por edad. Esto se puede realizar con varias columnas cuantitativas y poder obtener una información detallada. Por ahora realizaremos este proceso con las columnas: age, sex, last_month_salary, number_dependents, loan_type,  more_90_days_overdue, number_times_delayed_payment_loan_30_59_days y number_times_delayed_payment_loan_60_89_days.

4. **Identificar y manejar datos fuera del alcance del análisis.**

Según información otorgada para este punto, los organismos creen que usar la variable de género en los modelos de crédito (para decidir si se concede o aumenta un crédito) puede aumentar la desigualdad económica entre hombres y mujeres. Por eso, está prohibido usar el género en estas decisiones, ya que se considera discriminatorio. Así que no trabajaremos con la variable “sex” en nuestro proceso.

Crearemos una view donde uniremos las tablas para poder calcular las correlaciones:

```sql
CREATE OR REPLACE VIEW `proyecto-riesgo-relativo-3.prestamos.combined_view` AS
WITH combined_view AS (
    SELECT
    d.user_id,
	  d.default_flag,
	  ld.more_90_days_overdue,
    ld.using_lines_not_secured_personal_assets,
    ld.number_times_delayed_payment_loan_30_59_days,
    ld.debt_ratio,
    ld.number_times_delayed_payment_loan_60_89_days,
    lo.loan_id,
    lo.loan_type,
    ui.age,
    ui.last_month_salary,
    ui.number_dependents
FROM
    `proyecto-riesgo-relativo-3.prestamos.default` d
JOIN
    `proyecto-riesgo-relativo-3.prestamos.loans_detail` ld ON d.user_id = ld.user_id
JOIN
    `proyecto-riesgo-relativo-3.prestamos.user_info` ui ON d.user_id = ui.user_id
JOIN
    `proyecto-riesgo-relativo-3.prestamos.loans_outstanding` lo ON d.user_id = lo.user_id;
)
SELECT
    *
FROM
    combined_view
```

Calcularemos la correlación de ciertas variables para saber con cuáles trabajaremos. 

```sql
SELECT
  -- Correlaciones con default_flag
  CORR(default_flag, more_90_days_overdue) AS corr_default_more_90_days_overdue,
  CORR(default_flag, using_lines_not_secured_personal_assets) AS corr_default_using_lines,
  CORR(default_flag, number_times_delayed_payment_loan_30_59_days) AS corr_default_delay_30_59,
  CORR(default_flag, debt_ratio) AS corr_default_debt_ratio,
  CORR(default_flag, number_times_delayed_payment_loan_60_89_days) AS corr_default_delay_60_89,
  CORR(default_flag, age) AS corr_default_age,
  CORR(default_flag, last_month_salary) AS corr_default_salary,
  CORR(default_flag, number_dependents) AS corr_default_dependents,
  
  -- Correlaciones entre more_90_days_overdue y otras variables
  CORR(more_90_days_overdue, using_lines_not_secured_personal_assets) AS corr_90_days_using_lines,
  CORR(more_90_days_overdue, number_times_delayed_payment_loan_30_59_days) AS corr_90_days_delay_30_59,
  CORR(more_90_days_overdue, debt_ratio) AS corr_90_days_debt_ratio,
  CORR(more_90_days_overdue, number_times_delayed_payment_loan_60_89_days) AS corr_90_days_delay_60_89,
  CORR(more_90_days_overdue, age) AS corr_90_days_age,
  CORR(more_90_days_overdue, last_month_salary) AS corr_90_days_salary,
  CORR(more_90_days_overdue, number_dependents) AS corr_90_days_dependents,
  
  -- Correlaciones entre using_lines_not_secured_personal_assets y otras variables
  CORR(using_lines_not_secured_personal_assets, number_times_delayed_payment_loan_30_59_days) AS corr_using_lines_delay_30_59,
  CORR(using_lines_not_secured_personal_assets, debt_ratio) AS corr_using_lines_debt_ratio,
  CORR(using_lines_not_secured_personal_assets, number_times_delayed_payment_loan_60_89_days) AS corr_using_lines_delay_60_89,
  CORR(using_lines_not_secured_personal_assets, age) AS corr_using_lines_age,
  CORR(using_lines_not_secured_personal_assets, last_month_salary) AS corr_using_lines_salary,
  CORR(using_lines_not_secured_personal_assets, number_dependents) AS corr_using_lines_dependents,
  
  -- Correlaciones entre number_times_delayed_payment_loan_30_59_days y otras variables
  CORR(number_times_delayed_payment_loan_30_59_days, debt_ratio) AS corr_delay_30_59_debt_ratio,
  CORR(number_times_delayed_payment_loan_30_59_days, number_times_delayed_payment_loan_60_89_days) AS corr_delay_30_59_60_89,
  CORR(number_times_delayed_payment_loan_30_59_days, age) AS corr_delay_30_59_age,
  CORR(number_times_delayed_payment_loan_30_59_days, last_month_salary) AS corr_delay_30_59_salary,
  CORR(number_times_delayed_payment_loan_30_59_days, number_dependents) AS corr_delay_30_59_dependents,
  
  -- Correlaciones entre debt_ratio y otras variables
  CORR(debt_ratio, number_times_delayed_payment_loan_60_89_days) AS corr_debt_ratio_delay_60_89,
  CORR(debt_ratio, age) AS corr_debt_ratio_age,
  CORR(debt_ratio, last_month_salary) AS corr_debt_ratio_salary,
  CORR(debt_ratio, number_dependents) AS corr_debt_ratio_dependents,
  
  -- Correlaciones entre number_times_delayed_payment_loan_60_89_days y otras variables
  CORR(number_times_delayed_payment_loan_60_89_days, age) AS corr_delay_60_89_age,
  CORR(number_times_delayed_payment_loan_60_89_days, last_month_salary) AS corr_delay_60_89_salary,
  CORR(number_times_delayed_payment_loan_60_89_days, number_dependents) AS corr_delay_60_89_dependents,
  
  -- Correlaciones entre variables de perfil del usuario
  CORR(age, last_month_salary) AS corr_age_salary,
  CORR(age, number_dependents) AS corr_age_dependents,
  CORR(last_month_salary, number_dependents) AS corr_salary_dependents
FROM
  `proyecto-riesgo-relativo-3.prestamos.combined_view`;

```

En esta fórmula se calculan las correlaciones con todas las combinaciones posibles de las variables de la vista combined_view. Algunas correlaciones clave que se incluyen son:

- Correlaciones con default_flag: cómo se relaciona la variable de incumplimiento con otras variables.
- Correlaciones entre variables de mora (more_90_days_overdue, number_times_delayed_payment_loan_30_59_days, number_times_delayed_payment_loan_60_89_days): cómo se relacionan entre sí y con otras variables financieras.
- Correlaciones entre debt_ratio y otras variables financieras y demográficas.
- Correlaciones entre variables demográficas (age, last_month_salary, number_dependents): cómo se relacionan las características del usuario.

<aside>
💡 **Rango para Alta Correlación**
0.7 a 1.0 o -0.7 a -1.0: Se considera una alta correlación, ya sea positiva o negativa.

</aside>

**Resultados de correlaciones:**

- En Correlaciones con default_flag se obtuvo este resultado:

![Untitled (4)](https://github.com/user-attachments/assets/e209e357-9bb8-46b8-ae65-da134bb80939)

No existe alta correlación.

- En correlaciones entre more_90_days_overdue y otras variables se obtuvo este resultado:

![Untitled (5)](https://github.com/user-attachments/assets/85c66a1e-471d-4f3c-8db1-5e813acea440)

Sí existe alta correlación entre:

more_90_days_overdue - number_times_delayed_payment_loan_30_59_days

more_90_days_overdue - number_times_delayed_payment_loan_60_89_days

- En correlaciones entre using_lines_not_secured_personal_assets y otras variables se obtuvo este resultado:

![Untitled (6)](https://github.com/user-attachments/assets/5873bc3a-2e20-4580-af1e-88eba9c68603)

No existe alta correlación.

- En correlaciones entre number_times_delayed_payment_loan_30_59_days y otras variables se obtuvo este resultado:

![Untitled (7)](https://github.com/user-attachments/assets/1d2d2d67-c376-401c-96d8-32f8bec4c82c)

Sí existe alta correlación entre:

number_times_delayed_payment_loan_30_59_days y number_times_delayed_payment_loan_60_89_days

- En correlaciones entre debt_ratio y otras variables se obtuvo este resultado:

![Untitled (8)](https://github.com/user-attachments/assets/a9482edd-fcf5-43f8-9ef2-d30f84f2400a)

No existe alta correlación.

- En correlaciones entre number_times_delayed_payment_loan_60_89_days y otras variables se obtuvo este resultado:

![Untitled (9)](https://github.com/user-attachments/assets/7080b0eb-73d6-47ab-968e-975b58e6da7c)

No existe alta correlación.

- En correlaciones entre variables de perfil del usuario se obtuvo este resultado:

![Untitled (10)](https://github.com/user-attachments/assets/29486967-34e0-434c-9db9-df104f9ceeb8)

No existe alta correlación.

Estas son las variables que presentaron una correlación alta:

- more_90_days_overdue
- number_times_delayed_payment_loan_30_59_days
- number_times_delayed_payment_loan_60_89_days

Para tomar la decisión de cuál de estas variables se va a mantener, usaremos la desviación estándar.

```sql
SELECT
    STDDEV_SAMP(more_90_days_overdue) AS stddev_more_90_days_overdue,
    STDDEV_SAMP(number_times_delayed_payment_loan_30_59_days) AS stddev_30_59_days,
    STDDEV_SAMP(number_times_delayed_payment_loan_60_89_days) AS stddev_60_89_days
FROM 
    `proyecto-riesgo-relativo-3.prestamos.loans_detail`;
```

Se utiliza “STDDEV_SAMP” porque calcula la desviación estándar muestral, que es lo más común cuando se trabaja con muestras y no con la población completa.

**Resultados:**

![Untitled (11)](https://github.com/user-attachments/assets/05c672b4-0ce5-4a1c-a816-9584170c7c8e)

Las desviaciones estándar de las tres variables son bastante similares. Para decidir qué variable eliminar es importante considerar más que solo la desviación estándar y es basar nuestra decisión con criterios. En esta oportunidad conservaremos las 3 variables y avanzaremos.

5. **Identificar y manejar datos inconsistentes en variables categóricas.**

Las variables categóricas que tenemos son: **loan_type** y **default_flag**.

La variable loan_type presenta las siguientes clasificaciones: other, others y real estate.

Otra observación es que algunas palabras se encuentra totalmente en mayúsculas: OTHERS, REAL ESTATE.

Crearemos una nueva tabla donde tenga solo dos categorías y en minúsculas: other y real estate.

```sql
CREATE OR REPLACE TABLE `proyecto-riesgo-relativo-3.prestamos.loans_outstanding_clean` AS
SELECT 
  loan_id,
  user_id,
  LOWER(
    CASE 
      WHEN LOWER(loan_type) = 'others' THEN 'other'
      ELSE loan_type
    END
  ) AS loan_type
FROM `proyecto-riesgo-relativo-3.prestamos.loans_outstanding`;
```

Una vez realizado el cambio, debemos actualizar nuestra VIEW para tener los datos correspondientes:

```sql
CREATE OR REPLACE VIEW `proyecto-riesgo-relativo-3.prestamos.combined_view` AS
WITH combined_view AS (
    SELECT
        d.user_id,
        d.default_flag,
        ld.more_90_days_overdue,
        ld.using_lines_not_secured_personal_assets,
        ld.number_times_delayed_payment_loan_30_59_days,
        ld.debt_ratio,
        ld.number_times_delayed_payment_loan_60_89_days,
        lo.loan_id,
        lo.loan_type,
        ui.age,
        ui.last_month_salary,
        ui.number_dependents
    FROM
        `proyecto-riesgo-relativo-3.prestamos.default` d
    JOIN
        `proyecto-riesgo-relativo-3.prestamos.loans_detail` ld ON d.user_id = ld.user_id
    JOIN
        `proyecto-riesgo-relativo-3.prestamos.user_info` ui ON d.user_id = ui.user_id
    JOIN
        `proyecto-riesgo-relativo-3.prestamos.**loans_outstanding_clean**` lo ON d.user_id = lo.user_id
        
)
SELECT
    *
FROM
    combined_view
```

<aside>
💡 **INFORMACIÓN QUE PUEDE SERVIR**

</aside>

Como estamos trabajando con la versión gratuita del BigQuery, hay comandos que no nos permite utilizar. Ejemplo:

```sql
UPDATE proyecto-riesgo-relativo-3.prestamos.loans_outstanding
SET loan_type = LOWER(
CASE
WHEN LOWER(loan_type) = 'others' THEN 'other'
ELSE LOWER(loan_type)
END
)
```

con esta fórmula utilizando el comando UPDATE, se podía actualizar la variable de nuestra tabla original sin necesidad de crear otra tabla con los datos limpios. Como en nuestro caso no se puede hacer, se realizó el cambio como mostramos anteriormente paso a paso.

---

Para la tabla de *loans_outstanding_clean* tenemos duplicados los *user_id* porque se está reflejando la cantidad de préstamos que llevan:

![Untitled (12)](https://github.com/user-attachments/assets/6ad8169f-e114-43f9-ade4-adf931799701)

Aquí vemos reflejado varias filas con el *user_id* duplicado pero con un *loan_id* único ya que es el código del préstamo y el *loan_type* que es el tipo de préstamo. Crearemos otra tabla y se crearán las siguientes variables: *Real_Estate, Other* y *Total_Loans*. 

```sql
CREATE OR REPLACE TABLE `proyecto-riesgo-relativo-3.prestamos.loans_outstanding_ordenado` AS
SELECT
  user_id,
  COUNTIF(LOWER(loan_type) = 'real estate') AS Real_Estate,
  COUNTIF(LOWER(loan_type) = 'other') AS Other,
  COUNT(loan_id) AS Total_Loans
FROM
  `proyecto-riesgo-relativo-3.prestamos.loans_outstanding_clean`
GROUP BY
  user_id;
```

Se verá de esta manera:

![Untitled (13)](https://github.com/user-attachments/assets/e699a5a1-d4c7-458f-8aef-22f7096148fb)

Ahora se modificará la tabla view general para que tenga los datos actualizados:

```sql
CREATE OR REPLACE VIEW `proyecto-riesgo-relativo-3.prestamos.combined_view` AS
WITH combined_view AS (
    SELECT
        d.user_id,
        d.default_flag,
        ld.more_90_days_overdue,
        ld.using_lines_not_secured_personal_assets,
        ld.number_times_delayed_payment_loan_30_59_days,
        ld.debt_ratio,
        ld.number_times_delayed_payment_loan_60_89_days,
        lo.Real_Estate,
        lo.Other,
        lo.Total_Loans,
        ui.age,
        ui.last_month_salary,
        ui.number_dependents
    FROM
        `proyecto-riesgo-relativo-3.prestamos.default` d
    JOIN
        `proyecto-riesgo-relativo-3.prestamos.loans_detail` ld ON d.user_id = ld.user_id
    JOIN
        `proyecto-riesgo-relativo-3.prestamos.user_info` ui ON d.user_id = ui.user_id
    JOIN
        `proyecto-riesgo-relativo-3.prestamos.loans_outstanding_ordenado` lo ON d.user_id = lo.user_id
        
)
SELECT
    *
FROM
    combined_view
```

El resultado sería 35.575 usuarios a evaluar de 36.000 ya que 425 usuarios no tienen actividad de haber realizado un préstamo. Eso equivale a 1.18%, no es un porcentaje alto por ende descartaremos estos datos para nuestro análisis.

6. **Identificar y manejar datos discrepantes en variables numéricas.**

En este paso trabajaremos con los outliers, que son valores atípicos. Para identificarlos analizaremos los cuartiles y la distribución de los datos. Para calcular cuartiles en BigQuery, es importante elegir variables numéricas continuas o discretas que representen una distribución de interés. Las recomendadas para este caso son:

- **age:** Edad del cliente.
- **last_month_salary:** Último salario mensual reportado.
- **number_dependents:** Número de dependientes.
- **using_lines_not_secured_personal_assets**: Proporción de utilización del límite de crédito en líneas no garantizadas.
- **number_times_delayed_payment_loan_30_59_days**: Número de veces que el cliente se retrasó en el pago de un préstamo entre 30 y 59 días.
- **number_times_delayed_payment_loan_60_89_days:** Número de veces que el cliente se retrasó en el pago de un préstamo entre 60 y 89 días.
- **more_90_days_overdue**: Número de veces que el cliente estuvo más de 90 días vencido.
- **debt_ratio:** Relación de deuda del prestatario.

Cualquiera de estas variables es adecuada para un análisis de cuartiles, ya que representan datos cuantitativos sobre los cuales puedes medir tendencias de distribución. Los cuartiles son especialmente útiles para entender la dispersión y la centralidad de los datos, así como para identificar outliers. Trabajaremos con cada variable:

### Age

Identificamos los valores atípicos por medio de Big Query. Para la variable de *age* se planteó la siguiente fórmula:

```sql
SELECT
  user_id,
  age,
  PERCENTILE_CONT(age, 0.25) OVER() AS Q1,
  PERCENTILE_CONT(age, 0.50) OVER() AS Q2,  -- Cálculo de la mediana
  PERCENTILE_CONT(age, 0.75) OVER() AS Q3,
  (PERCENTILE_CONT(age, 0.75) OVER() - PERCENTILE_CONT(age, 0.25) OVER()) AS IQR,
  CASE
    WHEN age < (PERCENTILE_CONT(age, 0.25) OVER() - 1.5 * (PERCENTILE_CONT(age, 0.75) OVER() - PERCENTILE_CONT(age, 0.25) OVER()))
      OR age > (PERCENTILE_CONT(age, 0.75) OVER() + 1.5 * (PERCENTILE_CONT(age, 0.75) OVER() - PERCENTILE_CONT(age, 0.25) OVER()))
    THEN 'Outlier'
    ELSE 'Not Outlier'
  END AS outlier_flag
FROM
  `proyecto-riesgo-relativo-3.prestamos.combined_view`
```

![Untitled (14)](https://github.com/user-attachments/assets/9aeb9ac1-7588-4792-af35-9e759d3635e5)

Visualmente podemos ver los cuartiles, la mediana y si el usuario es o no un outlier.

Para tener una mejor visualización, abrimos el resultado en google sheets y filtramos cuáles son los usuarios outliers.

![Untitled (15)](https://github.com/user-attachments/assets/4b3e0072-a7b2-4604-a9fd-c0b6a7880b6b)

El total fue de 27 outliers que comprenden la edad de 95 en adelante. Si visualizamos esta variable por medio de una gráfica podemos notar también esa información:

![Untitled (16)](https://github.com/user-attachments/assets/1aa267fa-5c4f-405f-b595-688605cae31e)

Esta gráfica fue realizada en Power BI, así podemos ver qué datos son outliers. Al momento de crear el boxplot se agregaron dos valores:

![Untitled (17)](https://github.com/user-attachments/assets/15d1dde5-fdeb-41d4-a0a8-59379c532b24)

Y se utilizó python para poder obtener la gráfica:

```python
import pandas as pd
import matplotlib.pyplot as plt

# Crear un dataframe con los datos
df = dataset

# Crear el boxplot
plt.figure(figsize=(10, 6))  # Puedes ajustar el tamaño según tus necesidades
plt.boxplot(df['age'])
plt.title('Boxplot de la Edad')
plt.ylabel('Edad')
plt.grid(True)
plt.show()

```

### **last_month_salary**

Con esta variable tenemos una variedad de números por el cual identificar los outliers es algo más elaborado. Partimos con calcular los cuartiles en SQL:

```sql
SELECT
  user_id,
  last_month_salary,
  PERCENTILE_CONT(last_month_salary, 0.25) OVER() AS Q1,
  PERCENTILE_CONT(last_month_salary, 0.50) OVER() AS Q2,  -- Cálculo de la mediana
  PERCENTILE_CONT(last_month_salary, 0.75) OVER() AS Q3,
  (PERCENTILE_CONT(last_month_salary, 0.75) OVER() - PERCENTILE_CONT(last_month_salary, 0.25) OVER()) AS IQR,
  CASE
    WHEN last_month_salary < (PERCENTILE_CONT(last_month_salary, 0.25) OVER() - 1.5 * (PERCENTILE_CONT(last_month_salary, 0.75) OVER() - PERCENTILE_CONT(last_month_salary, 0.25) OVER()))
      OR last_month_salary > (PERCENTILE_CONT(last_month_salary, 0.75) OVER() + 1.5 * (PERCENTILE_CONT(last_month_salary, 0.75) OVER() - PERCENTILE_CONT(last_month_salary, 0.25) OVER()))
    THEN 'Outlier'
    ELSE 'Not Outlier'
  END AS outlier_flag
FROM
  `proyecto-riesgo-relativo-3.prestamos.combined_view`
```

Visualmente tenemos este resultado:

![Untitled (18)](https://github.com/user-attachments/assets/ca0cdf26-e93a-4759-be17-3cc17cee3bab)

Calculamos también el IQR que es el rango intercuartílico, es una medida de dispersión estadística que ayuda a entender la distribución de los datos y es especialmente útil para identificar valores atípicos. IQR es la resta de Q3 - Q1.

Esta fórmula nos mostraba como “not outliers” valores 0 y valores nulos, sin contar algunos números que no son considerados “normales” para el monto de *last_month_salary* como: 1, 4, 10, entre otros. Por esta razón se decidió aplicar un Winsorizing que es una técnica de transformación de datos utilizada en estadística para reducir el efecto de los valores atípicos. 

Se refiere a sustituir todos los valores por debajo del percentil 5 por el valor en el percentil 5; lo mismo aplica para los valores superior del percentil 95 que se reemplazan por el valor del percentil 95. Esto se usa para "recortar" los extremos del conjunto de datos, reduciendo la influencia de los valores atípicos en el análisis estadístico.

En este caso se calculará cuál es el valor del percentil 5 y los valores que sean menores se considerarán outliers. Con respecto a los valores nulos los clasificaremos como “Check Value” ya que de igual forma debemos evaluar otros valores de los clientes.

![Untitled (19)](https://github.com/user-attachments/assets/66b2530c-42af-44f3-a336-cb949b730ddf)

Ese es el valor de percentil 5 y percentil 95.

Crearemos una formula dónde se consideren todas las situaciones planteadas anteriormente:

```sql
WITH Percentiles AS (
  SELECT
    user_id,
    last_month_salary,
    PERCENTILE_CONT(last_month_salary, 0.25) OVER() AS Q1,
    PERCENTILE_CONT(last_month_salary, 0.50) OVER() AS Q2,
    PERCENTILE_CONT(last_month_salary, 0.75) OVER() AS Q3,
    PERCENTILE_CONT(last_month_salary, 0.05) OVER() AS p5,
    PERCENTILE_CONT(last_month_salary, 0.95) OVER() AS p95
  FROM
    `proyecto-riesgo-relativo-3.prestamos.combined_view`
)

SELECT
  user_id,
  last_month_salary,
  Q1,
  Q2,
  Q3,
  p5,
  p95,
  CASE
    WHEN last_month_salary IS NULL THEN 'Check Value'
    WHEN last_month_salary < p5 THEN 'Outlier'
    WHEN last_month_salary > p95 THEN 'Not Outlier'
    WHEN last_month_salary < Q1 - 1.5 * (Q3 - Q1) THEN 'Outlier'
    WHEN last_month_salary > Q3 + 1.5 * (Q3 - Q1) THEN 'very high salary'

  END AS outlier_flag
FROM
  Percentiles
```

Así tendremos: valores menos de 1300 son outliers, los valores superiores al promedio son very high salary, los normales son not outliers y los nulos son check value.

### **number_dependents**

Con este boxplot podemos ver que mayormente los usuarios tienen desde 0 hasta 2 cargas dependientes, de 3 en adelante son atípicos. Se mantendrán a medida que se vaya analizando.

![Untitled (20)](https://github.com/user-attachments/assets/48db69a6-f8ff-4fe8-ad7f-add0c17369e6)

Hay un total de 910 usuarios que aparecen como datos nulos, a estos se les asignará el número 0 para no tener fallas al momento de seguir avanzando en el análisis. De una vez también trabajaremos con la variable last_moth_salary (ya que se encuentran en la misma tabla) y los valores nulos les asignaremos el valor de la mediana; mientras que los valores que sean menores al percentil 5 tendrán el mismo valor de este que es 1.300 y los valores igual o superiores al percentil 95 tendrán el mismo valor de este que es 14.700.

Crearemos una nueva tabla con sus modificaciones con esta fórmula:

```sql
CREATE TABLE `proyecto-riesgo-relativo-3.prestamos.user_info2` AS
SELECT
  user_id,
  age,
  sex,
    CASE
    WHEN last_month_salary IS NULL THEN 5416 ------ mediana
    WHEN last_month_salary BETWEEN 0 AND 1300 THEN 1300
    WHEN last_month_salary >= 14700 THEN 14700
    ELSE last_month_salary
  END AS last_month_salary,
  IF(number_dependents IS NULL, 0, number_dependents) AS number_dependents
FROM
 `proyecto-riesgo-relativo-3.prestamos.user_info`
```

De esta manera tendremos esta tabla lista para seguir analizando.

### **using_lines_not_secured_personal_assets**

En esta parte veremos a los usuario ha excedido o no su límite de crédito asignado. Como los datos se reflejan en decimales, los transformaremos a % para hacer más sencillo el análisis.

```sql
SELECT
  user_id,
  FORMAT("%.2f%%", using_lines_not_secured_personal_assets * 100) AS using_lines_not_secured_personal_assets_percentage
FROM
  `proyecto-riesgo-relativo-3.prestamos.combined_view`;
```

Con esta fórmula todos los datos se reflejarán en %:

![image](https://github.com/user-attachments/assets/07d2e4f5-90be-4321-810d-4e8cf4b74e99)

Ahora especificaremos que los usuarios que superen el 100% son los que exceden su límite de crédito disponible en líneas no garantizadas. Significa que pueden afectar negativamente el puntaje de crédito del usuario ya que sería un riesgo darles un préstamo por como manejan sus finanzas y deben ser monitorizados.

```sql
SELECT
  user_id,
  FORMAT("%.2f%%", using_lines_not_secured_personal_assets * 100) AS using_lines_not_secured_personal_assets_percentage,
  CASE
    WHEN using_lines_not_secured_personal_assets > 1 THEN 'Monitorize'
    ELSE 'Normal'
  END AS monitoring_status
FROM
  `proyecto-riesgo-relativo-3.prestamos.combined_view`;
```

Se visualiza así:

![image (1)](https://github.com/user-attachments/assets/7fddb264-74ac-4840-af00-6e6f3ed6a85d)

Esto es solo una query que utilizamos para visualizar, pero más delante se creerá la variable.

### **number_times_delayed_payment_loan: 30, 59, 60, 89 y 90 more days**

Se realizó un boxplot con las 3 variables juntas, de ahí podemos visualizar los outliers:

![image (2)](https://github.com/user-attachments/assets/3169aa07-4d84-48bd-ad31-bc5ab2a303a2)

se hizo en Power BI con Python, usando esta fórmula:

```python
# Importar las bibliotecas necesarias
import pandas as pd
import matplotlib.pyplot as plt

# Asegúrate de que tu DataFrame se llama 'dataset'
# Power BI automáticamente pasará los datos a la variable 'dataset'

# Configurar el gráfico
plt.figure(figsize=(12, 6))

# Creando un boxplot para more_90_days_overdue
plt.subplot(1, 3, 1)  # 1 fila, 3 columnas, primer gráfico
plt.boxplot(dataset['more_90_days_overdue'].dropna())  # Elimina los valores nulos
plt.title('Días de Mora > 90 Días')
plt.ylabel('Cantidad de Días')

# Creando un boxplot para number_times_delayed_payment_loan_30_59_days
plt.subplot(1, 3, 2)  # 1 fila, 3 columnas, segundo gráfico
plt.boxplot(dataset['number_times_delayed_payment_loan_30_59_days'].dropna())  # Elimina los valores nulos
plt.title('Retrasos de Pago (30-59 Días)')
plt.ylabel('Cantidad de Retrasos')

# Creando un boxplot para number_times_delayed_payment_loan_60_89_days
plt.subplot(1, 3, 3)  # 1 fila, 3 columnas, tercer gráfico
plt.boxplot(dataset['number_times_delayed_payment_loan_60_89_days'].dropna())  # Elimina los valores nulos
plt.title('Retrasos de Pago (60-89 Días)')
plt.ylabel('Cantidad de Retrasos')

# Mostrar el gráfico
plt.tight_layout()
plt.show()
```

Más adelante descartaremos el outlier que está presente en las 3 variables cerca del 100.

### **debt_ratio**

En esta parte veremos  la relación entre las deudas totales del usuario y su patrimonio o ingresos totales. Este indicador es muy común en las evaluaciones financieras, especialmente en el análisis de crédito, ya que proporciona una medida de la capacidad de un prestatario para gestionar y pagar sus deudas con los recursos económicos que tiene disponibles.

Como los datos se reflejan en decimales, los transformaremos a % para hacer más sencillo el análisis. Los usuarios que superen el 100% indican un nivel alto de deuda en relación con los activos o ingresos, lo que puede ser un indicador de problemas financieros potenciales y un riesgo más alto para quienes otorgan créditos.

```sql
SELECT
  user_id,
  FORMAT("%.2f%%", debt_ratio * 100) AS debt_ratio_percentage,
  CASE
    WHEN debt_ratio > 1 THEN 'Monitorize'
    ELSE 'Normal'
  END AS monitoring_status
FROM
  `proyecto-riesgo-relativo-3.prestamos.combined_view`;
```

Con esta fórmula en BigQuery podemos saber qué usuarios están en los estándares normales y quienes tienen un nivel alto de deuda.

![image (3)](https://github.com/user-attachments/assets/21a73937-8e97-4c71-8416-5229fa84c404)

Esto es solo una query que utilizamos para visualizar, pero más delante se creerá la variable.

7. **Crear nuevas variables.**

Durante el proceso creamos las variables *Real_Estate, Other* y *Total_Loans*, ahora queremos crear una nueva variable sobre el número de veces que el cliente se retrasó en el pago de un préstamo. Contamos con 3 variables para eso:

- more_90_days_overdue
- number_times_delayed_payment_loan_30_59_days
- number_times_delayed_payment_loan_60_89_days

Queremos una nueva que trate la ponderación de retrasos para darle más importancia a los retrasos más largos, para eso se asigna un valor diferente a cada tipo de retraso. Por ejemplo, los retrasos de 30-59 días se ponderan con 1, los de 60-89 días con 2, y los de más de 90 días con 3, reflejando su mayor gravedad.

```sql
SELECT 
    user_id,
    (1 * number_times_delayed_payment_loan_30_59_days + 2 * number_times_delayed_payment_loan_60_89_days + 3 * more_90_days_overdue) AS weighted_delays
FROM 
    `proyecto-riesgo-relativo-3.prestamos.combined_view`
```

Teniendo esta fórmula podemos también tener la oportunidad de crear las variables de % de: debt_ratio y using_lines_not_secured_personal_assets.

```sql
SELECT 
    user_id,
    (1 * number_times_delayed_payment_loan_30_59_days + 2 * number_times_delayed_payment_loan_60_89_days + 3 * more_90_days_overdue) AS weighted_delays,
    
    FORMAT("%.2f%%", debt_ratio * 100) AS debt_ratio_percentage,
    CASE
        WHEN debt_ratio > 1 THEN 'Monitorize'
        ELSE 'Normal'
    END AS monitoring_status_debt_ratio,
    
    FORMAT("%.2f%%", using_lines_not_secured_personal_assets * 100) AS lines_not_secured_percentage,
    CASE
        WHEN using_lines_not_secured_personal_assets > 1 THEN 'Monitorize'
        ELSE 'Normal'
    END AS monitoring_status_lines_not_secured
    
FROM 
    `proyecto-riesgo-relativo-3.prestamos.combined_view`;
```

Crearemos una nueva tabla:

```sql
CREATE TABLE `proyecto-riesgo-relativo-3.prestamos.nuevas_variables` AS
SELECT 
    user_id,
    (1 * number_times_delayed_payment_loan_30_59_days + 2 * number_times_delayed_payment_loan_60_89_days + 3 * more_90_days_overdue) AS weighted_delays,
    
    FORMAT("%.2f%%", debt_ratio * 100) AS debt_ratio_percentage,
    CASE
        WHEN debt_ratio > 1 THEN 'Monitorize'
        ELSE 'Normal'
    END AS monitoring_status_debt_ratio,
    
    FORMAT("%.2f%%", using_lines_not_secured_personal_assets * 100) AS lines_not_secured_percentage,
    CASE
        WHEN using_lines_not_secured_personal_assets > 1 THEN 'Monitorize'
        ELSE 'Normal'
    END AS monitoring_status_lines_not_secured
    
FROM 
    `proyecto-riesgo-relativo-3.prestamos.combined_view`;
```

Visualización de la nueva tabla:

![image (4)](https://github.com/user-attachments/assets/f8c26dd0-66d6-4cd8-ab6e-f92821c8f402)

Crearemos unas variables categóricas para *age*, *Total_Loans, last_month_salary y weighted_delays* por medio de los cuartiles:

```sql
CREATE TABLE `proyecto-riesgo-relativo-3.prestamos.nuevas_variables_categoricas` AS
WITH cuartiles AS (
  SELECT
    user_id,
    age,
    last_month_salary,
    Total_Loans,
    weighted_delays
  FROM
    `proyecto-riesgo-relativo-3.prestamos.combined_view`
)

SELECT
  user_id,
  age,
  last_month_salary,
  Total_Loans,
  weighted_delays,
  
  CASE
    WHEN age BETWEEN 18 AND 39 THEN 'ADULTO JOVEN'
    WHEN age BETWEEN 40 AND 64 THEN 'ADULTO MEDIO'
    WHEN age BETWEEN 65 AND 74 THEN 'ADULTO MAYOR'
    ELSE 'SENIOR'
  END AS cat_grupo_etario,

  CASE
    WHEN Total_Loans BETWEEN 0 AND 4 THEN 'BAJOS'
    WHEN Total_Loans BETWEEN 5 AND 7 THEN 'MODERADO'
    WHEN Total_Loans BETWEEN 8 AND 10 THEN 'ALTO'
    ELSE 'MUY ALTO'
  END AS cat_total_prestamos,

  CASE
    WHEN weighted_delays = 0 THEN 'SIN MORA'
    WHEN weighted_delays BETWEEN 1 AND 2 THEN 'BAJO'
    WHEN weighted_delays BETWEEN 3 AND 5 THEN 'MODERADO'
    WHEN weighted_delays BETWEEN 6 AND 8 THEN 'ALTO'
    ELSE 'MUY ALTO'
  END AS cat_weighted_delays,

  CASE
    WHEN last_month_salary BETWEEN 1300 AND 3900 THEN 'BAJOS'
    WHEN last_month_salary BETWEEN 3901 AND 5400 THEN 'MODERADO'
    WHEN last_month_salary BETWEEN 5401 AND 7499 THEN 'ALTO'
    ELSE 'MUY ALTO'
  END AS cat_salary
FROM
  cuartiles

```

Aquí los categorizamos y creamos una nueva tabla

---

Modificaremos la variable *default_flag* ya que se notó que al tener dos morosidades algún cliente lo muestra como buen pagador (0) por ende haremos que cuente a todos los clientes desde su primera morosidad como mal pagadores (1)

```sql
CREATE TABLE `proyecto-riesgo-relativo-3.prestamos.default_modificado` AS
SELECT
user_id,
number_times_delayed_payment_loan_30_59_days,
number_times_delayed_payment_loan_60_89_days,
more_90_days_overdue,
  CASE
    WHEN number_times_delayed_payment_loan_30_59_days > 0
      OR number_times_delayed_payment_loan_60_89_days > 0
      OR more_90_days_overdue > 0
    THEN 1
    ELSE 0
  END AS default_flag_actualizado
FROM `proyecto-riesgo-relativo-3.prestamos.combined_view`;
```

Con esto se solucionará el problema.

8. **Unir tablas.**

Modificando nuestra view que habíamos creado anteriormente, agregaremos las nuevas modificaciones, variables y ordenaremos:

```sql
WITH combined_view AS (
    SELECT
        d.user_id,
        ui.age,
        vc.cat_grupo_etario,
        ui.last_month_salary,
        vc.cat_salary,
        ui.number_dependents,
        d.default_flag_actualizado,
        ld.using_lines_not_secured_personal_assets,
        nv.lines_not_secured_percentage,
        nv.monitoring_status_lines_not_secured,
        ld.number_times_delayed_payment_loan_30_59_days,
        ld.number_times_delayed_payment_loan_60_89_days,
        ld.more_90_days_overdue,
        nv.weighted_delays,
        vc.cat_weighted_delays,
        ld.debt_ratio,
        nv.debt_ratio_percentage,
        nv.monitoring_status_debt_ratio,
        lo.Real_Estate,
        lo.Other,
        lo.Total_Loans,
        vc.cat_total_prestamos
        
    FROM
        `proyecto-riesgo-relativo-3.prestamos.default_modificado` d
    JOIN
        `proyecto-riesgo-relativo-3.prestamos.loans_detail` ld ON d.user_id = ld.user_id
    JOIN
        `proyecto-riesgo-relativo-3.prestamos.user_info2` ui ON d.user_id = ui.user_id
    JOIN
        `proyecto-riesgo-relativo-3.prestamos.loans_outstanding_ordenado` lo ON d.user_id = lo.user_id
    JOIN
        `proyecto-riesgo-relativo-3.prestamos.nuevas_variables` nv ON d.user_id = nv.user_id
    JOIN
        `proyecto-riesgo-relativo-3.prestamos.nuevas_variables_categoricas` vc ON d.user_id = vc.user_id
        
)
SELECT
    *
FROM
    combined_view
WHERE
    user_id != 21979
```

Agregaremos también la exclusión de un user_id que no es necesario tener en la información ya que es un outlier muy notorio. Es el outlier que mencionamos que está cerca de los 100 días de mora e las 3 variables.

Aquí parte de la visualización (no está completa):

![image (5)](https://github.com/user-attachments/assets/a2b7ec98-e9ba-4cb7-9ea5-ca28824f3d95)

