Imagine a reader that has followed your previous experiences with Spark. 

The goal of this post is to share your work on the Commoncrawl data with them, emphasizing the problems of working with unstructured Web data.

Readers will be curious to learn basic statistics about samples taken from the crawl
but also how long it takes to make a pass over (subsets of) the data. 
Ideally, your analysis teaches us something about the Web we did not yet know; 
but that is not a requirement for completing the assignment.



## Testing setup
Before we start running our spark program on the large crawl dataset we first download a segment to test and debug the code on. Therefore I randomly took a  segment of the May 2019 crawl:
```
crawl-data/CC-MAIN-2019-22/segments/1558232255092.55/warc/CC-MAIN-20190519181530-20190519203530-00268.warc.gz
```

## Setup
Load the WARC file:
```scala
val warcfile = "/opt/docker/data/CC-MAIN-20190519181530-20190519203530-00268.warc.gz"
```

```scala
val warcf = sc.newAPIHadoopFile(
              warcfile,
              classOf[WarcInputFormat],               // InputFormat
              classOf[LongWritable],                  // Key
              classOf[WarcRecord]                     // Value
    )
```

## First Goal of this project
Find Steam game keys

```scala
import java.io.InputStreamReader;
def getContent(record: WarcRecord):String = {
  val cLen = record.header.contentLength.toInt
  //val cStream = record.getPayload.getInputStreamComplete()
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
```

```scala
import java.io.IOException;
import org.jsoup.Jsoup;
def HTML2Txt(content: String) = {
  try {
    Jsoup.parse(content).text().replaceAll("[\\r\\n]+", " ")
  }
  catch {
    case e: Exception => throw new IOException("Caught exception processing input row ", e)
  }
}
```
```scala
val warcc = warcf.
  filter{ _._2.header.warcTypeIdx == 2 /* response */ }.
  filter{ _._2.getHttpHeader().contentType != null }.
  filter{ _._2.getHttpHeader().contentType.startsWith("text/html") }.
  map{wr => ( wr._2.header.warcTargetUriStr, HTML2Txt(getContent(wr._2)) )}.cache()
```
```scala
val contains = warcc.map{tt => StringUtils.substring(tt._2, 0, 999999999).toLowerCase()}.
  filter{x => x contains "steam"}.
  filter{x => x contains "game"}
```

Lets check how many sites have the words `game` and `steam` in them:
```scala
contains.count()
```
Returns
```scala
302
```
Hmm, this is bad news, only 302 sites in this segment have content about steam games. This will make it hard for testing. But who knows maybe we are lucky, to find the steam codes we first need to know what the format is. After googeling around the format seemst to be:
```
xxxxx-xxxxx-xxxxx
or
xxxxx-xxxxx-xxxxx-xxxxx
```
Where x can be a number, lower case character or upper case character. Just to be sure I wil also check:
```
xxxxx-xxxxx-xxxxx-xxxxx
```
These combinations can be checked with a regular expression (regex), in Scala this can be done with a library, lets make a regular expression and test it:
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
This looks correct, lets implement it as a whole.

```scala
val contains = warcc.map{tt => StringUtils.substring(tt._2, 0, 128).toLowerCase()}.
  filter{x => x contains "steam"}.
  filter{x => x contains "game"}.
filter{x => !(pattern1 findAllIn x).mkString(",").trim.isEmpty || !(pattern2 findAllIn x).mkString(",").trim.isEmpty || !(pattern3 findAllIn x).mkString(",").trim.isEmpty}

contains.count()
```
Returns
```
0
```
Aah, no free steam codes :( , this is not too surprising if we thing about how often these would appear and how big of a segment of the internet we have scanned. For this project to work we should scan a significant part of the CommonCrawl, this is not feasable on one computer with a normal internet connection. Up to the next challenge!

## New goal



### To get a feeling of the data lets have a look at 





