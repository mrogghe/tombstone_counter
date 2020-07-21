
---
***NOTE***: 

1). This is a re-mavenized version of dse6_gradle with support for same versions and includes some minor typo corrections and updated instructions for compiling and executing.

2). This is a new branch (dse6_gradle) that is designed to work with DSE version 6.0+. 
    
3). The original branch (ossc3_dse5) was developed to only work with OSS C* 3.x versions, including compatible DSE 5.0.x and 5.1.x versions.

4). Neither branch is designed working with older version of C* (version 2.x and before)

In my test, the utility has been built with DSE 6.7.8 libraries and it has been tested working with all the following DSE and C* versions:
* DSE 6.7.x (6.7.8)
* OSS C* 3.11.x (3.11.4)

**IMPORTANT**: 

This repo (branch) only provides the utility source code, but NOT the depending DSE libraries. If you want to use this utility, please make sure you **follow DataStax license requirements and policies**!

---

# Overview

Cassandra database (including DataStax Enterprise - DSE) uses an immutable file structure, called SSTable, to store data physically on disk. With such an append-only structure, "tombstone" is needed to mark the deletion of a piece of data in Cassandra (aka, soft deletion). The downside of a "tombstone", though, is that it will slow down the read performance because Cassandra still needs to scan all tombstoned data in order to return the correct result and this process conusmes computer resources. 

Cassandra evicts out tombstones through the "compaction" process. But in order to avoid some edge cases about data correctness (e.g. to avoid zombie data resurrection), tombstones need to stay in Cassandra for at least gc_grace_seconds (default 10 days). So if the age of a tombstone is less than gc_grace_seconds, the compaction process will not touch it. This means that if not properly planned (from data modeling and application development perspective), there could have execessive amount of tombstones in Cassandra system and therefore the overall read performance could be suffered.

From my field experience with Cassandra and DSE consulting, it is not an uncommon situation to see too many tombstones in a Cassandra/DSE system. When this happens, a frequent question from clients is "how can I find out how many tombstones are there in the system?". The "sstablemetadata" Cassandra tool can partially (and indirectly) answer that question by giving out an estimated droppable tombstone time period/count table (see example below) for a single SSTable. 

```
Estimated tombstone drop times:
1510712520:         1
1510712580:         1
1510712640:         1
```

In order to get the total amount of tombstones in the system for a Cassandra table, you have to sum the droppable tombstone counts for all time periods and then repeat the process for all SSTables for that Cassandra table. This process is a little bit cumbersome and more importantly, it can't tell you the number of tombstones of different kinds. In lieu of this, I write a tool (as presented in this repository) to help find tombstone counts for a Cassandra table, both in total and at different category levels.

# Compiling the Tool

This utility works with Apache OSS Cassandra as is.  mvn clean install or mvn clean package

To work with DSE Cassandra:

Within the pom.xml, in the following section, change addClasspath to false

```
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.1.2</version>
        <configuration>
          <archive>
            <manifest>
              <addClasspath>false</addClasspath>
```

This instruction causes the built MANIFEST to not contain any classpath information, allowing us to set that at execution time.

Download the desired version of the DSE Cassandra database (not the driver, the database itself).  Unpack/install and locate the DSE Cassandra libraries.
 
* The location of the DSE C* library jar files are in the following location:
  * For packaged installation: /usr/share/dse/cassandra/lib
  * For tarball installation: <tarball_install_home>/resources/cassandra/lib 
  
The location should have many jar files including one that looks similar to:  dse-db-all-6.7.8.jar

Note this directory path for next section.

# Using the Tool

To execute the program for Apache OSS Cassandra, run the following command (assuming under the same directory where the jar file is):
```
java -cp tombstone_counter-1.0.jar com.castools.TombStoneCounter <options>
```
To execute the program for DSE Cassandra, run the following command (assuming under the same directory where the jar file is) using the path from previous section:
```
java -cp "./tombstone_counter-1.0.jar:/usr/local/cassandra/dse-6.7.8/resources/cassandra/lib/*" com.castools.TombStoneCounter -d <options>
```
On Windows the : should be a ;

The supported program options are as below.
```
usage: TombStoneCounter [-d <arg>] [-h] [-o <arg>] [-sp]

SSTable TombStone Counter for Apache Cassandra 3.x
Options:
  -d,--dir <arg>    Specify Cassandra table data directory
  -h,--help         Displays this help message.
  -o,--output <arg> Specify output file for tombstone stats.
  -sp,--suppress    Suppress commandline display output details. Only the basic info is displayed. 
```
When "-d" or "-o" option is not specified, it will use the current directory as the default directory for Cassandra table data (SSTable files) directory and the output diretory for generated tombstone statistics file.

If "-sp" option is specified, it will not display tombstone detail information on the command-line output while processing each SSTable.

## Output

When running the tool with specified Cassandra table data directory ("-d") option, the tool will generate a tombstone statistics (csv) file that includes the tombstone count of various categories for each SSTable file. Meanwhile, it will also prints out more read-able information on the console output. Below is an example (against DSE 6.0.10):

```
$ java -cp tombstone_counter-1.0.jar com.castools.TombStoneCounter -d test.bkup.6010/testbl-fa44b341176a11eaa91a83381c121464/

Processing SSTable data files under directory: test.bkup.6010/testbl-fa44b341176a11eaa91a83381c121464/

  ------------------------------------------------------------------------
  (scanning files, it may take long time to finish for large size of data)
  ------------------------------------------------------------------------

  Analyzing SSTable File: aa-1-bti-Data.db
      Total partition: 8
      Tomstone Count (Total): 4
      Tomstone Count (Partition): 3
      Tomstone Count (Range): 0
      Tomstone Count (ComplexColumn): 0
      Tomstone Count (Row) - Deletion: 0
      Tomstone Count (Row) - TTL: 0
      Tomstone Count (Cell) - Deletion: 1
      Tomstone Count (Cell) - TTL: 0
```

Meanwhile, a tomstone statistics file (**tombstone_stats.csv**) is generated in the current directory with the following contents. This statsitics file can be easily further analyzed using other tools like Excel
```
$ cat tombstone_stats.csv
sstable_data_file,part_cnt,total_ts_cnt,ts_part_cnt,ts_range_cnt,ts_complexcol_cnt,ts_row_del_cnt,ts_row_ttl_cnt,ts_cell_del_cnt,ts_cell_ttl_cnt
aa-1-bti-Data.db,8,4,3,0,0,0,0,1,0
```


# Future Improvements

Currently this utility is single-threaded and scans SSTables on one DSE node (for a particular C* table) sequentially. For C* tables with lots of data, it may take a long time to complete the scan. This can be improved with the following 2 possible options:
1. Allow to scan multiple SSTables concurrently.
2. For one particular SSTable, allow to use multiple theads to scan it.

Another improvement that can make the utility a little easier to use is:

3. Instead of providing the actual file system folder name (that contains the SSTable files to be scanned), the utility can simply take C* keyspace and table name as the input parammeters.
