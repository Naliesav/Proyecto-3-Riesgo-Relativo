# Proyecto-3-Riesgo-Relativo

# Ficha Técnica Proyecto 3: Riesgo relativo

- **Objetivo:**
    
    Evaluar el riesgo relativo para clasificar clientes solicitantes por probabilidad de incumplimiento. 
    
- **Equipo:**
Individual.
- **Herramientas y Tecnologías:**

SQL. 

BigQuery.

Looker Studio.

https://docs.google.com/spreadsheets/u/0/

https://chatgpt.com/

https://www.youtube.com/

### **Procesar y preparar base de datos:**

### **Procesamiento y análisis:**

1. **Conectar/importar datos a otras herramientas.**
    1. Descargar y descomprimir data set.
    2. Subir archivos a BigQuery en formato CSV.

![prueba](https://github.com/user-attachments/assets/31204295-de28-4e29-b244-f4218d8bc2e8)


1. **Identificar y manejar valores nulos.**

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
