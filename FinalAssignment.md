# Final project Big Data
### Sam Sweere (s4403142)

In this blog we are going to work with the common crawl dataset. This dataset contains every website that could be found on the internet gathered by a crawler. We are going to work with the lastest crawl from May 2019. This crawl contains 2.65 billion web pages that after formating still account to 220 TiB of data.

## Goal of the project
This blog will consits out of two parts:

 - Finding Steam game keys in webpages.
 - Visualizing the location where hosting servers reside in the world.

The reason for two goals is that the first goal was relatively simple and not too interesting, therefore I decided to define a new objective.

# Finding Steam game keys
Steam is an online game selling and managing platform. When a game is sold outside of the online store or is passed on as a gift these games have to be activated using a game key. One can imagine that someone would by accident or as a giveaway place these gamekeys somewhere on the internet. Our goal is to find these game keys.

## Testing setup
Before we start running our spark program on the large crawl dataset we first have to think of a way to test the progam. Since we only have acces to a normal pc and the crawl contains 220 TiB of data we have to use a part of the data. Luckaly the commoncrawl team thought of this and split the whole crawl in 56000 seperate files. The file we will use to test and debug on is a randomly picked segment of the May 2019 crawl:
```
crawl-data/CC-MAIN-2019-22/segments/1558232255092.55/warc/CC-MAIN-20190519181530-20190519203530-00268.warc.gz
```

## WARC, WAT and WET
The commoncrawl formated the data into WARC, WAT and WET files. The WARC format are the raw data from the crawls, containing the website data (excluding images) and all other metadata. The WAT files contain only the metadata, like URL, IP adress, responce codes, etc... Finally the WET files only contain extracted plaintext. 

## Setup WARC file
Since we want to find find the steam game keys in the content of the web page we are going to use the WARC files, first load the downloaded WARC file:
```scala
val warcfile = "/opt/docker/data/CC-MAIN-20190519181530-20190519203530-00268.warc.gz"
```
Next we format the data using the surfsara `WarcInputFormat` package: 
```scala
val warcf = sc.newAPIHadoopFile(
              warcfile,
              classOf[WarcInputFormat],               // InputFormat
              classOf[LongWritable],                  // Key
              classOf[WarcRecord]                     // Value
    )
```

## Convert HTML to Text using Jsoup
To make it easier to scan the websites content that is written in HTML we want to convert it to text. Jsoup is a library that is widly used to acchieve this. The code how this is done is not too interesting and originates from the example, added for completeness.
```scala
import java.io.InputStreamReader;
import java.io.IOException;
import org.jsoup.Jsoup;

def getContent(record: WarcRecord):String = {
  val cLen = record.header.contentLength.toInt
  val cStream = record.getPayload.getInputStream()
  val content = new java.io.ByteArrayOutputStream();

  val buf = new Array[Byte](cLen)
  
  var nRead = cStream.read(buf)
  while (nRead != -1) {
    content.write(buf, 0, nRead)
    nRead = cStream.read(buf)
  }

  cStream.close()
  
  content.toString("UTF-8");
}

def HTML2Txt(content: String) = {
  try {
    Jsoup.parse(content).text().replaceAll("[\\r\\n]+", " ")
  }
  catch {
    case e: Exception => throw new IOException("Caught exception processing input row ", e)
  }
}
```
Next we create and RDD containing all plane text of websites that contain content:
```scala
val warcc = warcf.
  filter{ _._2.header.warcTypeIdx == 2 /* response */ }.
  filter{ _._2.getHttpHeader().contentType != null }.
  filter{ _._2.getHttpHeader().contentType.startsWith("text/html") }.
  map{wr => ( wr._2.header.warcTargetUriStr, HTML2Txt(getContent(wr._2)) )}.cache()
```
## Finding the game keys
To find the game keys we need to do two things:
- Filter sites that talk about `steam` in combination with `game`.
- Look for potential gamekeys.

Let's first have a look how many sites in our segment of the commoncrawl contain (parts) of the words `steam` and `game`. We do this by first converting everything to lower case, such that we don't miss the words if they are capitalized and next filter on the words:
```scala
val contains = warcc.map{x => x._2.toLowerCase()}.
  filter{x => x contains "steam"}.
  filter{x => x contains "game"}
```
Lets see how many hits we have:
```scala
contains.count()
```
Returns:
```scala
302
```
Hmm, this is potentially bad news, only 302 sites in this segment have content about steam games. This will make it hard for testing. But who knows maybe we are lucky, to find the steam codes themselfs we first need to know what the format is. After googeling around the format seems to be:
```
xxxxx-xxxxx-xxxxx
or
xxxxx-xxxxx-xxxxx-xxxxx
```
Where x can be a number, lower case character or upper case character. Just to be sure I wil also check:
```
xxxxx-xxxxx-xxxxx-xxxxx
```
These combinations can be checked with a regular expression (regex), in Scala this can be done with a library. Since our gamekeys can be a number, lower case character or upper case character we will use the regex command `[a-zA-Z0-9]`. Let's make the regular expression to find the gamekeys and test it:
```scala
import scala.util.matching.Regex

val pattern1 = new Regex("[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]-[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]-[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]")
val pattern2 = new Regex("[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]-[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]-[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]-[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]")
val pattern3 = new Regex("[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]-[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]-[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]-[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]-[a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9][a-zA-Z0-9]")


val test1 = "12345-abcde-FGHIJ-1k4ld-EjfL4"
val test2 = "123456-abcde-FGHIJ-1k4ld"
val test3 = "123456-abcdef-FGHIJ-1k4ld-EjfL4"

println("Pattern1, test1: " + (pattern1 findAllIn test1).mkString(","))
println("Pattern1, test2: " + (pattern1 findAllIn test2).mkString(","))
println("Pattern1, test3: " + (pattern1 findAllIn test3).mkString(","))

println("Pattern2, test1: " + (pattern2 findAllIn test1).mkString(","))
println("Pattern2, test2: " + (pattern2 findAllIn test2).mkString(","))
println("Pattern2, test1: " + (pattern2 findAllIn test3).mkString(","))

println("Pattern3, test1: " + (pattern3 findAllIn test1).mkString(","))
println("Pattern3, test2: " + (pattern3 findAllIn test2).mkString(","))
println("Pattern3, test3: " + (pattern3 findAllIn test3).mkString(","))
```

This returns:
```
Pattern1, test1: 12345-abcde-FGHIJ
Pattern1, test2: 23456-abcde-FGHIJ
Pattern1, test3: bcdef-FGHIJ-1k4ld
Pattern2, test1: 12345-abcde-FGHIJ-1k4ld
Pattern2, test2: 23456-abcde-FGHIJ-1k4ld
Pattern2, test1: bcdef-FGHIJ-1k4ld-EjfL4
Pattern3, test1: 12345-abcde-FGHIJ-1k4ld-EjfL4
Pattern3, test2: 
Pattern3, test3: 
```
This looks to be correct, let's now check how many sites have a potential game key in them:
```scala
val contains = warcc.map{tt => StringUtils.substring(tt._2, 0, 128).toLowerCase()}.
  filter{x => x contains "steam"}.
  filter{x => x contains "game"}.
filter{x => !(pattern1 findAllIn x).mkString(",").trim.isEmpty || !(pattern2 findAllIn x).mkString(",").trim.isEmpty || !(pattern3 findAllIn x).mkString(",").trim.isEmpty}

contains.count()
```
Returns:
```
0
```
To bad, no free steam codes in this segment. This is not too surprising if we thing about how often these would appear and how big of a segment of the internet we have scanned (1/56000). For this project to work we should scan a significant part of the CommonCrawl, this is not feasable on one computer with a normal internet connection. We could try to make it a standalone application and scan a bigger part of the commoncrawl, but I think we can find more interesting information from the dataset. Let's see this as a warmup for a harder challenge!

# Visualizing the location where hosting servers reside in the world
Evey website needs a hosting server, this server has a IP address at which it can be reached. For IP addresses there is a way to roughly estimate their location in the world. The goal of this challange is to map these locations.

## General pipline
To acchieve this the following steps have to be done:
- Extract the IP address and top level domain from the commoncrawl. 
- Check if we alrady encountered the IP, if so skip.
- Find the geo-location of the IP adress.
- Generate a map of these geo-location counts.
- Bonus: find a correlation between top level domain (.nl, .com, etc.. ) and where they are hosted.


## Retrieving the commoncrawl segments
In this challenge we only need the IP address and the URL, not the contens of the websites. Thus in this case the WAT format is ideal, this is at the same time also a bit smaller size (+- 300 MiB per segment instead of +- 900 MiB) making the calculations a bit faster. In order to be able to scan a bigger part of the commoncrawl dataset we will include the retrieving of the segments in the code. The idea is that every iteration the Spark program will download a segment, retrieve the locations, detelete the segment and downlaod the next segment. When looking at the first two segments of the May 2019 crawl one could think we could iterate over the last number:
```
crawl-data/CC-MAIN-2019-22/segments/1558232254253.31/wat/CC-MAIN-20190519061520-20190519083520-00000.warc.wat.gz

and

crawl-data/CC-MAIN-2019-22/segments/1558232254253.31/wat/CC-MAIN-20190519061520-20190519083520-00001.warc.wat.gz
```
But this is not the case since the path is longer see:
```
crawl-data/CC-MAIN-2019-22/segments/1558232254253.31/warc/CC-MAIN-20190519061520-20190519083520-00000.warc.gz

and

crawl-data/CC-MAIN-2019-22/segments/1558232254731.5/warc/CC-MAIN-20190519081519-20190519103519-00000.warc.gz
```
Both end with `00000`. The fix is to load the WAT path file that is available on the commoncrawl site.
Lets start with creating a function that can download the files:
```scala
def fileDownloader(url: String, filename: String) = {
    new URL(url) #> new File(filename) !!
}
```
Download the WAT paths and put every line into a array such that we can iterate over the array:
```scala
//Get the segment paths
val watPathsUrl = "https://commoncrawl.s3.amazonaws.com/crawl-data/CC-MAIN-2019-22/wat.paths.gz"

fileDownloader(watPathsUrl, "wat.paths.gz")

var in = new GZIPInputStream(new FileInputStream("wat.paths.gz"))
var watSegments = ArrayBuffer[String]()

//Save every line as an element of the array
for (line <- Source.fromInputStream(in).getLines()) {
        watSegments += line
}
```
For testing purposes lets download the first segment:
```scala
val segNum = 0
val commonCrawlUrl = "https://commoncrawl.s3.amazonaws.com/" + watSegments(segNum)

//Replace / with - to prevent errors in filenames
val watfile = watSegments(segNum).replace('/', '-')

//Download the segment file
fileDownloader(commonCrawlUrl, watfile)
```
Initialy I used the surfsara `WarcInputFormat` package as a did in the previous part, however when we check for IP addresses we encounter some problems:
```scala
warc.map{wr => wr._2.header}.filter{_.warcIpAddress.length() != 0}.map{x => (x.warcInetAddress, x.warcIpAddress)}.take(100)
```
Crashes for some reason, and:
```scala
warc.map{wr => wr._2.header}.map{x => (x.warcInetAddress, x.warcIpAddress)}.take(100)
```
Returns:
```
res41: Array[(java.net.InetAddress, String)] = Array((null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (nu...
```
Are the IP addresses not in the data? This seems weird, lets check the raw WAT data:
```
WARC/1.0
WARC-Type: metadata
WARC-Target-URI: http://004-ford-f-2.wiring-diagram.klymburn.co.uk/post/save-your-ears-8211-a-noise-meter-circuit
WARC-Date: 2019-05-27T15:39:42Z
WARC-Record-ID: <urn:uuid:f6bc04e9-e10a-463d-8b19-2082060e1120>
WARC-Refers-To: <urn:uuid:43771f4f-1f87-468e-baab-6fc750ef35d1>
Content-Type: application/json
Content-Length: 1490

{"Container":{"Filename":"CC-MAIN-20190519061520-20190519083520-00000.warc.gz","Compressed":true,"Offset":"479","Gzip-Metadata":{"Inflated-Length":"740","Footer-Length":"8","Inflated-CRC":"631744713","Deflate-Length":"470","Header-Length":"10"}},"Envelope":{"Format":"WARC","WARC-Header-Length":"415","Actual-Content-Length":"321","WARC-Header-Metadata":{"WARC-IP-Address":"128.199.40.232","WARC-Target-URI":"http://004-ford-f-2.wiring-diagram.klymburn.co.uk/post/save-your-ears-8211-a-noise-meter-circuit", ...
```
I clearly see IP addresses, but I used all the IP commands the `WarcHeader` has. When investigating more I could not find any documentation on the commands possible for `WarcInputFormat`, so after a few frustrating hours I decided to do the parsing myself. The seccond part of the raw WAT data is saved in the JSON format. Lets first filter the WAT data to only have the JSON parts

We need to start a sql SparkSession to read a file:
```scala
import org.apache.spark.sql.SparkSession
val spark: SparkSession = SparkSession.builder.master("local").getOrCreate
```
We can now read the WAT file, since we only want the json part we filter on lines that start with `"{\"Container\":"`, where the `\"` are used to escape the chars, otherwise Scala would think the String would stop at `"`. Next parse the JSON files:
```scala
//Get all the json files and parse them
val watJsonStr = spark.read.textFile(watfile).filter(x => x.startsWith("{\"Container\":"))
val wat = spark.read.json(watJsonStr).cache()
```
To check if we where succesfull we can check the schema:
```scala
wat.printSchema
```
Returns:
```
root
 |-- Container: struct (nullable = true)
 |    |-- Compressed: boolean (nullable = true)
 |    |-- Filename: string (nullable = true)
 |    |-- Gzip-Metadata: struct (nullable = true)
 |    |    |-- Deflate-Length: string (nullable = true)
 |    |    |-- Footer-Length: string (nullable = true)
 |    |    |-- Header-Length: string (nullable = true)
 |    |    |-- Inflated-CRC: string (nullable = true)
 |    |    |-- Inflated-Length: string (nullable = true)
 |    |-- Offset: string (nullable = true)
 |-- Envelope: struct (nullable = true)
 |    |-- Actual-Content-Length: string (nullable = true)
 |    |-- Block-Digest: string (nullable = true)
 |    |-- Format: string (nullable = true)
 ...
```
Good news, we have parsed data!

## Retrieving the URL and IP address
We now need to get the URL and IP-Address from the json files. We do not care if the server gives a response, if the crawler found an address and the DNS returned an ip address then the domain exists which is good enough for locating the servers. Therefore we are going to extract the URL and IP-Address from the requests. We do not want the IP field or URL to be null:

```scala
//Select the url and ip address
val urlIp = wat.select($"Envelope.WARC-Header-Metadata.WARC-Target-URI".as("URL"),$"Envelope.WARC-Header-Metadata.WARC-IP-Address".as("IP-Address")).
            filter($"URL".isNotNull && $"IP-Address".isNotNull)
```
Lets check what we end up with:
```scala
urlIp.show()
```
Returns:
```
+--------------------+---------------+
|                 URL|     IP-Address|
+--------------------+---------------+
|http://004-ford-f...| 128.199.40.232|
|http://004-ford-f...| 128.199.40.232|
|http://004-ford-f...| 128.199.40.232|
|http://004-ford-f...| 128.199.40.232|
|http://007dingjin...|156.234.166.118|
|http://007dingjin...|156.234.166.118|
|http://008795.cn/...| 134.73.253.219|
|http://008795.cn/...| 134.73.253.219|
...
```
Looks good!

For our analysis we want to have the IP-Address and the top level domain (nl, com, uk, etc..). The top level domain can be extracted from the URL, we fist extract the hostname from the url and then split the restuling hostname on a "." and take the last element:

```scala
val selection = urlIp.withColumn("host", callUDF("parse_url", $"URL", lit("HOST")))
    .select($"IP-Address", substring_index($"host", ".", -1).as("topLevelDomain"))
```
Lets see what results:
```scala
selection.show
```
Returns:
```
+---------------+--------------+
|     IP-Address|topLevelDomain|
+---------------+--------------+
| 128.199.40.232|            uk|
| 128.199.40.232|            uk|
| 128.199.40.232|            uk|
| 128.199.40.232|            uk|
|156.234.166.118|           com|
|156.234.166.118|           com|
...
```
We got the first step done!

## Detecting duplicate IP
There are two parts on detecting duplicate IP addresses:
- Within one segment: Here we are going to use `groupBy` in combination with an aggregation function.
- Over multplie segments: Since we are going to analyse a lot of IP-Addresses on different segments maintaining an array to track the already analysed IP-Addresses will be to memory intesive. Therefore we are going to use bloom filter. One downside of bloomfilters is the chance of false positives, but since we are not doing something exact an estamation with some fasle positives is good enough.

### Within one segment
Here we use the `groupBy` in combination with the `agg` function. We pick the first top level domain since they are (if everthing went right) the same on one IP address, therefore `first()` takes the least ammount of calculation:
```scala
val uniqueIp = selection.groupBy("IP-Address").agg(first("topLevelDomain"))
```

### Over multiple segments
Bloom filter MMM

## Geolocation
To determine the country of the ip adress I downloaded the IP to country dataset from MaxMind [link](https://dev.maxmind.com/geoip/geoip2/geolite2/). Since we want the spark file to be indepentend we download the file from github and load id as a dataframe. We are only intersted in the `network` and `geoname_id` columns:
```scala
fileDownloader("https://raw.githubusercontent.com/rubigdata/cc-2019-SamSweere/master/GeoLite2-Country-CSV_20190625/GeoLite2-Country-Blocks-IPv4.csv?token=AIZE7NT7RUVTXMILYDKYOT25D73SI", 
               "GeoLite2-Country-Blocks-IPv4.csv")

val ipLoc = spark.read.format("csv").option("header", "true")
  .load("GeoLite2-Country-Blocks-IPv4.csv").select("network","geoname_id")
```
`ipLoc` now contains:
```
+------------+----------+
|     network|geoname_id|
+------------+----------+
|  1.0.0.0/24|   2077456|
|  1.0.1.0/24|   1814991|
|  1.0.2.0/23|   1814991|
|  1.0.4.0/22|   2077456|
|  1.0.8.0/21|   1814991|
| 1.0.16.0/20|   1861060|
| 1.0.32.0/19|   1814991|
...
```
Next we need to transfer the geoname_id to the countries, luckaly MaxMind included the translation file:
```scala
fileDownloader("https://raw.githubusercontent.com/rubigdata/cc-2019-SamSweere/master/GeoLite2-Country-CSV_20190625/GeoLite2-Country-Locations-en.csv?token=AIZE7NXTP67TCC2EUEPVVP25D742I",
               "GeoLite2-Country-Locations-en.csv")

val locName = spark.read.format("csv").option("header", "true")
  .load("GeoLite2-Country-Locations-en.csv").select("geoname_id","country_iso_code","country_name")
```
Know we can combine these two into one ip to country dataframe:
```scala
val ipCountry = ipLoc.join(locName, Seq("geoname_id")).
  select("network","country_iso_code","country_name").cache()
```


## Problems
Adding geolibraries didnt work, could not get the import working. Fixed it by downloading the dataset.


`warc.take(10)` crashes the kernel, very impractical.


There is no place where is stored what the warcTypeIdx number map to, very bad documentation. Found it by

```scala
warc.map{wr => wr._2.header}.map{x => x.warcTypeIdx}.take(10)
```

```scala
warc.map{wr => wr._2.header}.filter{_.warcIpAddress.length() != 0}.map{x => (x.warcInetAddress, x.warcIpAddress)}.take(100)
```
Crashes for some reason



```scala
warc.map{wr => wr._2.header}.map{x => (x.warcInetAddress, x.warcIpAddress)}.take(100)
```
Returns:
```
res41: Array[(java.net.InetAddress, String)] = Array((null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (null,null), (nu...
```
I know the information is there thus this is not working.

Conclusion, this is not working, lets write my own parser

