# Assignment 3

## Mission
In this research I will look at the statistical dataset until 2017 from the province of Nijmegen (https://opendata.nijmegen.nl/dataset/statistische-data-tot-2017).

## Setup
First we need to make spark session and load the csv data. The dataset orignaly does not come in the csv format, you can use Microsoft Excel to transform it to an csv file.
```scala
import org.apache.spark.sql.types._ 

val spark = SparkSession
  .builder()
  .appName("A3-Nijmegen-Statistics-spark-df")
  .config("spark.executor.memory", "3G")
  .getOrCreate()

val std = spark.read.format("csv").option("header", true).load("/data/bigdata/StadsgetallenNijmegen.csv").cache()
```
We can now look at the structure of the data using. 
```scala
std.printSchema
std.show(5)
```
This returns the following structure.
```
root
 |-- WaardeId: string (nullable = true)
 |-- Waarde: string (nullable = true)
 |-- WaardetypeNaam: string (nullable = true)
 |-- ThemaNaam: string (nullable = true)
 |-- OnderwerpNaam: string (nullable = true)
 |-- Labelgroepnaam: string (nullable = true)
 |-- LabelNaam: string (nullable = true)
 |-- GeografieType: string (nullable = true)
 |-- GeografieOmschrijving: string (nullable = true)
 |-- TijdOmschrijving: string (nullable = true)
 |-- TijdType: string (nullable = true)
 |-- BronOrganisatie: string (nullable = true)
 |-- BronNaam: string (nullable = true)

+--------+------+----------------+---------+--------------------+--------------------+-----------+-------------+---------------------+----------------+---------+-----------------+--------------------+
|WaardeId|Waarde|  WaardetypeNaam|ThemaNaam|       OnderwerpNaam|      Labelgroepnaam|  LabelNaam|GeografieType|GeografieOmschrijving|TijdOmschrijving| TijdType|  BronOrganisatie|            BronNaam|
+--------+------+----------------+---------+--------------------+--------------------+-----------+-------------+---------------------+----------------+---------+-----------------+--------------------+
| 1141740|   124|Absolute waarden|Bevolking|Geslacht en leeftijd|geslacht naar lee...|  man 13-18|         Wijk|              Tolhuis|        1-1-2006|Peildatum|Gemeente Nijmegen|Gemeentelijke Bas...|
| 1141741|   105|Absolute waarden|Bevolking|Geslacht en leeftijd|geslacht naar lee...|vrouw 13-18|         Wijk|              Tolhuis|        1-1-2006|Peildatum|Gemeente Nijmegen|Gemeentelijke Bas...|
| 1141742|   126|Absolute waarden|Bevolking|Geslacht en leeftijd|geslacht naar lee...|  man 19-24|         Wijk|              Tolhuis|        1-1-2006|Peildatum|Gemeente Nijmegen|Gemeentelijke Bas...|
| 1141743|   116|Absolute waarden|Bevolking|Geslacht en leeftijd|geslacht naar lee...|vrouw 19-24|         Wijk|              Tolhuis|        1-1-2006|Peildatum|Gemeente Nijmegen|Gemeentelijke Bas...|
| 1141744|   311|Absolute waarden|Bevolking|Geslacht en leeftijd|geslacht naar lee...|   man 0-12|         Wijk|           Zwanenveld|        1-1-2006|Peildatum|Gemeente Nijmegen|Gemeentelijke Bas...|
+--------+------+----------------+---------+--------------------+--------------------+-----------+-------------+---------------------+----------------+---------+-----------------+--------------------+
only showing top 5 rows

```
There are a few things to note here. First all the datatypes are strings while clearly the `WaardeId` and `Waarde` probably don't have to be strings. We can also observe that this dataset uses labels for different subjects. Lets have a look what kind of subjects this dataset contains. We can see the top 10 most contained subjest using the following code.

```scala
val subjects = std.groupBy("OnderwerpNaam").count()
subjects.orderBy(desc("count")).show(10)
```

```
+--------------------+------+
|       OnderwerpNaam| count|
+--------------------+------+
|          Etniciteit|152040|
|Geslacht en leeftijd| 96564|
|            Migratie| 33712|
|      Woningvoorraad| 25591|
| Geboorte en sterfte| 23912|
|Uitkeringen, Wet ...| 17510|
|Niet-werkende wer...| 12932|
|   Woningen, verkoop| 12376|
|Woningen, verhuur...| 11660|
|     Arbeidsplaatsen|  7280|
+--------------------+------+
```
It is clear that the subject are verry diverse, from etnicity to non-working job seekers.




Where is the lowest unemployment


This will be done using the following datasets:
* BAG dataset (https://www.nijmegen.nl/opendata/BAG_ADRES.csv)

