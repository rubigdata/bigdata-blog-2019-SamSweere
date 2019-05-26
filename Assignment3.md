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
There are a few things to note here. First all the datatypes are strings while clearly the `WaardeId` and `Waarde` probably don't have to be strings. We can also observe that this dataset uses labels for different subjects. 
Lets have a look what kind of subjects this dataset contains. We can see the top 10 most contained subjest using the following code.

```scala
val subjects = std.groupBy("OnderwerpNaam").count()
subjects.orderBy(desc("count")).show(10)
```
Results:

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

## Goal
Before we start working with the dataset more we first need some goals for this project in increasing difficulty:
  * In which neighborhood in Nijmegen do most children younger than 10 live?
  * What is the richest neighboorhood in Nijmegen based on income?
  * Is there a correlation between the income of a neighboorhood and the ammount of public artworks within that nieghboorhood?

## Preprocessing the database 



The dataset is structured around a set of subjects, these subject apply to specific neighoorhoods in Nijmegen in a specific year. First we need to preprocess the database such that it is easer to work with. In the structure it is shown that there is a column called `TijdType` (Time type) and `TijdOmschrijving` (Time description) let see what they mean. By running:

```scala
std.groupBy("TijdType").count().show
```
We get:
```
+------------+------+
|    TijdType| count|
+------------+------+
|Kalenderjaar|119825|
|  Schooljaar|  1080|
|   Peildatum|306580|
+------------+------+
```
There are thus three time types, lets see what kind of time description format is assosiated with them. This is done by first selecting the columns we are interested in. Next we group them by `TijdType`. Finally we take the first value that is assosiated with the `TijdType`.
```scala
std.select('TijdType, 'TijdOmschrijving).groupBy("TijdType").agg(first("TijdOmschrijving")).show
```
Results in.
```
+------------+------------------------------+
|    TijdType|first(TijdOmschrijving, false)|
+------------+------------------------------+
|Kalenderjaar|                          2007|
|  Schooljaar|                     2002-2003|
|   Peildatum|                      1-1-2006|
+------------+------------------------------+
```
Thus `Kalenderjaar` has a year, `Schooljaar` has a range between years and `Peildatum` has a specific date and they are all saved in a string format.

First we are going to look at the `Schooljaar` time type, since there are only 1080 entries in the database of that type. 
```scala
std.filter("TijdType = 'Schooljaar'").groupBy("OnderwerpNaam").count().show
```
Results
```
+-------------+-----+
|OnderwerpNaam|count|
+-------------+-----+
| Huursubsidie| 1080|
+-------------+-----+
```
The subject `Huursubsidie` is not interesting for our research. Thus to make things easier we can eliminate all the time types of type `Schooljaar`. Next lets have a look at time type `Peildatum`:

```scala
std.filter("TijdType = 'Peildatum'").groupBy("OnderwerpNaam").count().show
```
Results
```
+--------------------+------+
|       OnderwerpNaam| count|
+--------------------+------+
|         Huishoudens|  6048|
|              Omvang|  1501|
|          Etniciteit|152040|
|     Woningbezetting|  1228|
|Uitkeringen, Wet ...| 17510|
|     Woningdichtheid|   446|
|         Huisvesting|  4480|
|Geslacht en leeftijd| 96564|
|Niet-werkende wer...| 12932|
|      Woningvoorraad| 13831|
+--------------------+------+
```
Since this time type contains the information of the age of citicens (this is where we are interested in) we cannot exclude this information. Thus we need to transform the `TijdOmschrijving` to be in the format of of only a year. We are going to cut corners here a bit and define that the year on the date is the year of the information. 

Next we filter the database on entries that are not interesting for our research:

```scala
//Remove time type "Schooljaar"
val noSchoolyear = std.filter("TijdType != 'Schooljaar'")

//Remove entries with null as value in "Waarde"
val noNullinVal = noSchoolyear.filter("Waarde is not null")

//Remove entries with null as value in "TijdOmschrijving"
val noNullInTime = noNullinVal.filter("TijdOmschrijving is not null")

//We only want neighboorhoods
val onlyNeigh = noNullinVal.filter("GeografieType = 'Wijk'")
```
Convert the strings to float and ints where needed and only select the columns that are interesting for our research:
```scala
// Registering a user-defined function that converts dates to years in Int form
val toYearInt = udf((x: String) => x.split('-').last.toInt)

// Registering a user-defined function that converts values to flaot form
val valueToFloat = udf((x: String) => x.toFloat)

val nijmData = onlyNeigh.select(valueToFloat($"Waarde") as "value",
                            $"OnderwerpNaam" as "subject",
                            $"Labelgroepnaam" as "group",
                            $"LabelNaam" as "label",
                            $"GeografieOmschrijving" as "quarter",
                            toYearInt($"TijdOmschrijving") as "year")
```
Lets see if this transformation resulted in a format we can work easier with:
```scala
nijmData.printSchema
nijmData.describe().show
```
Results
```
root
 |-- value: float (nullable = false)
 |-- subject: string (nullable = true)
 |-- group: string (nullable = true)
 |-- label: string (nullable = true)
 |-- quarter: string (nullable = true)
 |-- year: integer (nullable = false)

+-------+-----------------+--------------+--------------------+--------------------+----------+------------------+
|summary|            value|       subject|               group|               label|   quarter|              year|
+-------+-----------------+--------------+--------------------+--------------------+----------+------------------+
|  count|           328224|        328224|              328224|              328224|    328224|            328224|
|   mean|555.6452526713215|          null|                null|                null|      null|2004.7513100809203|
| stddev|7345.646648823478|          null|                null|                null|      null| 4.644832996373014|
|    min|           -183.0|        Omvang|             inkomen| niet-westers  ma...|  't Acker|              1995|
|    max|        551938.44|Woningvoorraad|zelfstandige woni...|  ? 500.001 en hoger|Zwanenveld|              2012|
+-------+-----------------+--------------+--------------------+--------------------+----------+------------------+
```
This looks perfect!



This will be done using the following datasets:
* BAG dataset (https://www.nijmegen.nl/opendata/BAG_ADRES.csv)

