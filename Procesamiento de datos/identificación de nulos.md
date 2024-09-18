# Identificar y manejar valores nulos.

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

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/4f6e4884-28e6-47a8-8b41-9105cb8c0fed/a9d1ba17-a703-4b3d-b9a7-f3ba318f91a0/Untitled.png)

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

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/4f6e4884-28e6-47a8-8b41-9105cb8c0fed/a673de8c-2530-434c-bb8d-f59b081d8077/Untitled.png)

Aquí nos refleja que 683 usuarios son mal pagadores. Tendremos este dato guardado para profundizar más adelante.
