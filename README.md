# Spark RetComCom
Retcomcom is little retention, compression and compaction tool written with the Spark framework to clean up and organise data on a hadoop cluster.

With huge amounts of data handled and processed on a HDFS cluster - file compaction, data compression format and the retention period for the data needs manged.

## Usage

**File this in when complete**


## Compaction
When streaming data into HDFS, small messages are written to a large number of files, if left unchecked this will cause unnecessary strain on the HDFS Name Node. To handle this, it is good practice to run a compaction job on directories that contain many small files. This helps reduce the resource strain of the Name Node by ensuring HDFS blocks are filled efficiently. It is common practice to do this type of compaction with MapReduce or on Hive tables / partitions. The RetComCom tool provides this functionality using Spark and Scala.

## Compression
At a high level the RetComCom tool will calculate the number of output files to efficiently fill the default HDFS block size on the cluster taking into consideration the size of the data, compression type, and serialization type.



**Compression Math - ratio assumptions**
```vim
SNAPPY_RATIO = 1.7;  // (100 / 1.7) = 58.8 ~ 40% compression rate on text 
LZO_RATIO = 2.0;     // (100 / 2.0) = 50.0 ~ 50% compression rate on text 
GZIP_RATIO = 2.5;    // (100 / 2.5) = 40.0 ~ 60% compression rate on text 
BZ2_RATIO = 3.33;    // (100 / 3.3) = 30.3 ~ 70% compression rate on text 
AVRO_RATIO = 1.6;    // (100 / 1.6) = 62.5 ~ 40% compression rate on text 
PARQUET_RATIO = 2.0; // (100 / 2.0) = 50.0 ~ 50% compression rate on text  
```
**Compression Ratio Formulas**  
```vim
Input Compression Ratio * Input Serialization Ratio * Input File Size = Input File Size Inflated  
Input File Size Inflated / ( Output Compression Ratio * Output Serialization Ratio ) = Output File Size  
Output File Size / Block Size of Output Directory = Number of Blocks Filled  
FLOOR( Number of Blocks Filled ) + 1 = Efficient Number of Files to Store  
```
#### Snappy
Snappy is intended to be fast, it compresses at about 250 MB/sec or more and decompresses at about 500 MB/sec or more. Typical compression ratios (based on googles benchmark suite) are about 1.5-1.7x for plain text. More sophisticated algorithms are capable of achieving yet higher compression rates, although usually at the expense of speed. Snappy is an ideal middle ground to compress data efficiently and quickly decompress the data to be read.
#### GZIP
GZIP is naturally supported by Hadoop, it’s based on the [DEFLATE](https://en.wikipedia.org/wiki/DEFLATE) algorithm, which is combination of [L277](https://en.wikipedia.org/wiki/LZ77_and_LZ78) and [Huffman Coding](https://en.wikipedia.org/wiki/Huffman_coding). GZIP has a higher compression ratio that Snappy (2.5x for plain text although this is at the expense of its read speed). GZIP is suitable as often there is a need to store the data for longer retention periods. If the amount of output per day is extensive, and if that data is required in the future, then the accumulated data will take extensive amount of HDFS space. This data may not be used very frequently, resulting in a waste of HDFS space. Therefore, it’s necessary to compress the output before storing long term on HDFS. 
#### Note: Compression splitting and Compaction
When considering how to compress data that will be processed by Spark, it is important to understand whether or not the compression format supports splitting. Consider an uncompressed file stored in HDFS of size 1 GB. With a default block size of 128 MB, the file will be stored as 7 blocks, and a Spark job using this file will create 7 input splits, each processed independently and input to a separate task.
Consider now a file is GZIP-compressed of size 1 GB. As before, HDFS will store the file as 7 blocks. However, creating a split for each block won’t work since it is impossible to start reading an arbitrary point in a GZIP stream and therefore impossible for a Spark task to read its split independently of others.  
In this case, Spark will do the right thing and not try and split the gzipped file, since it knows that the input is gzip-compressed (by looking at the filename extension) and that GZIP does not support splitting. This will work, but at the expense of a single node that will process the 7 HDFS blocks, most of which will not be local to that node. If this large file is compressed then the file cannot be split over different nodes thus only being able to be processed by a single node (effectively destroying the advantage of running a cluster of parallel machines).

###### For this reason, when re-writing the data from the SNAPPY codec to its new GZIP codec, it’s important that compaction is included to eliminate the problem that non-splitable compression can introduce.


## Retention
Setting a retention period for data (i.e. 100 days) can be useful to clean up old data or comply with data retention policies set by your organisation. 

## License
MIT [Licence](LICENCE.md)

###### Big thanks to KeitSmith who created a compaction Java project which inspired this tool - https://github.com/KeithSSmith/spark-compaction