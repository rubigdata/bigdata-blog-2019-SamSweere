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
  * In which neighborhood in Nijmegen do most youth (age <= 18) live in the year 2012?
  * What is the richest neighboorhood in Nijmegen based on income?

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
Results in:
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
Results:
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
Results:
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
Results:
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

## Kids
Lets tackle our first goal: In which neighborhood in Nijmegen do most youth (age <= 18) live in the year 2012?

First we can filter on the year:
```scala
val only2012 = nijmData.filter("year = 2012")
```
Next we have to find the `subject` we want to use. In this case we want the subject to involve age ("leeftijd" in Dutch).
```scala
val containsAge = nijmData.filter($"subject".contains("leeftijd") || $"subject".contains("Leeftijd"))
containsAge.groupBy("subject").count().show()
```
Results
```
+--------------------+-----+
|             subject|count|
+--------------------+-----+
|Geslacht en leeftijd|77760|
+--------------------+-----+
```
Looks like we got the right subject. Next lets look at the groups within this subject.
```scala
containsAge.groupBy("group").count().show()
```
Results
```
+--------------------+-----+
|               group|count|
+--------------------+-----+
|geslacht naar lee...| 8100|
|      leeftijd (3kl)|  270|
|geslacht naar lee...|29160|
|geslacht naar lee...| 4860|
|leeftijd (jeugd_5kl)| 4050|
|            geslacht| 1620|
|geslacht naar lee...| 8100|
|geslacht naar lee...|  540|
|     leeftijd (18kl)|14580|
|leeftijd (jeugd_3kl)| 2430|
|      leeftijd (5kl)| 4050|
+--------------------+-----+
```
Good news! The database as entries specificly for the youth ("jeugd" in Dutch) lets see what is in the group `leeftijd (jeugd_5kl)`.
```scala
val ageYouth = only2012.filter("group = 'leeftijd (jeugd_5kl)'")
ageYouth.groupBy("label").count()
```
Results
```
+-----+-----+
|label|count|
+-----+-----+
|13-18|   45|
|  2-3|   45|
|19-24|   45|
| 4-12|   45|
|  0-1|   45|
+-----+-----+
```
This looks like what we want to have. To be sure lets take a sample.
```scala
ageYouth.sample(true,0.06).show()
```
Returns
```
+-----+--------------------+--------------------+-----+--------------------+----+
|value|             subject|               group|label|             quarter|year|
+-----+--------------------+--------------------+-----+--------------------+----+
|  2.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)|  0-1|    Ooyse Schependom|2012|
|105.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)|  0-1|           Grootstal|2012|
|  0.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)|  0-1|Haven- industriet...|2012|
|138.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)|  0-1|             De Kamp|2012|
|  2.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)|  0-1|              Ressen|2012|
| 99.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)|  2-3|          Zwanenveld|2012|
|430.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)| 4-12|           Nije Veld|2012|
|324.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)| 4-12|           Meijhorst|2012|
|343.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)| 4-12|           Weezenhof|2012|
|595.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)| 4-12|            't Acker|2012|
|155.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)|13-18|             Goffert|2012|
|197.0|Geslacht en leeftijd|leeftijd (jeugd_5kl)|19-24|            Aldenhof|2012|
+-----+--------------------+--------------------+-----+--------------------+----+
```
Lets see where the most youth lives, by first filtering out the `19-24` label. Then grouping by the `quarter` taking the sum of all the youth living in that `quarter`. Finally we show the outcome in descending order:
```scala
val youthQuarters = ageYouth.filter("label != '19-24'").groupBy("quarter").agg(sum("value"))
.withColumnRenamed("sum(value)","count")
youthQuarters.orderBy(desc("count")).show(10)
```
Returns
```
+--------------+------+
|       quarter| count|
+--------------+------+
|    Oosterhout|2156.0|
|          Lent|1758.0|
|        Hatert|1544.0|
|Neerbosch-Oost|1483.0|
|      't Acker|1332.0|
|       De Kamp|1252.0|
|     Hengstdal|1251.0|
|     Hazenkamp|1246.0|
|      Heseveld|1186.0|
|     Wolfskuil|1123.0|
+--------------+------+
```
Thus in 2012 in the neighboorhood of Oosterhout (next to Lent) the most youth was living.

## Wealth
Our seccond goal was: What is the richest neighboorhood in Nijmegen based on income?

In this chapter I will not explain every step I took but only show the querry that I used to get to the awnser. The method is exaclty the same as the previous chapter.

Lets first see what years have income data:
```scala
nijmData.filter($"subject".contains("Inkomen")).groupBy("year").count().show()
```
Returns:
```
+----+-----+
|year|count|
+----+-----+
|2007| 1454|
|2009| 1440|
|2008| 1437|
+----+-----+
```
Okay, thus only a limited choise of years. Let's pick 2009 as the year.

After doing some research on the different `subjects`,`group` and `label` to choose from I decided to pick the average personal income per neighbourhood. 

```scala
var incomeQuarters = nijmData.filter("year = 2009")
                             .filter("subject = 'Inkomen, algemeen'")
                             .filter("label = 'inwoners (bedrag)'")
                             .groupBy("quarter")
                             .agg(sum("value"))
                             .withColumnRenamed("sum(value)", "Average income")

incomeQuarters.orderBy(desc("Average income")).show(10)
```
Results:
```
+------------+--------------+
|     quarter|Average income|
+------------+--------------+
| Kwakkenberg|       29600.0|
|  Hunnerberg|       29300.0|
|   Hazenkamp|       26100.0|
|   Weezenhof|       25900.0|
|  Galgenveld|       24900.0|
|  Oosterhout|       23600.0|
|     Altrade|       23000.0|
|    St. Anna|       22700.0|
|Brakkenstein|       22500.0|
|        Hees|       22500.0|
+------------+--------------+
```
Thus the richest neighboorhood in Nijmegen is Kwakkenberg.