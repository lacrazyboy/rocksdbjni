# LevelDB JNI

## Description

LevelDB JNI gives you a Java interface to the 
[LevelDB](http://code.google.com/p/leveldb/) C++ library
which is a fast key-value storage library written at Google 
that provides an ordered mapping from string keys to string values.. 

## Building Prerequisites 

* GNU compiler toolchain
* [Maven 3](http://maven.apache.org/download.html)

## Building

The following worked for me on:

 * OS X Lion with X Code 4
 * Ubuntu 10.04 (32 and 64 bit)

Then download the snappy, leveldb, and leveldbjni project source code:

    wget http://snappy.googlecode.com/files/snappy-1.0.3.tar.gz
    tar -zxvf snappy-1.0.3.tar.gz
    svn checkout http://leveldb.googlecode.com/svn/trunk/ leveldb
    git clone git://github.com/fusesource/leveldbjni.git

Compile the snappy project.  This produces a static library.    

    cd snappy-1.0.3 
    ./configure --disable-shared --with-pic
    make
    
Patch and Compile the leveldb project.  This produces a static library.    
    
    cd ../leveldb
    export LIBRARY_PATH=`cd ../snappy-1.0.3; pwd`
    export C_INCLUDE_PATH=${LIBRARY_PATH}
    export CPLUS_INCLUDE_PATH=${LIBRARY_PATH}
    patch -p 0 < ../leveldbjni/leveldb.patch
    make

Now use maven to build the leveldbjni project.    
    
    cd ../leveldbjni
    mvn clean install -P download -Dleveldb=`cd ../leveldb; pwd` -Dsnappy=`cd ../snappy-1.0.3; pwd`

The above will produce:

* `target/leveldbjni-${version}.jar` : The java class file to the library.
* `target/leveldbjni-${version}-native-src.zip` : A GNU style source project which you can use to build the native library on other systems.
* `target/leveldbjni-${version}-${platform}.jar` : A jar file containing the built native library using your currently platform.

## Using as a Maven Dependency

You just nee to add the following repositories and dependencies to your Maven pom.

    <repositories>
      <repository>
        <id>fusesource.nexus.snapshot</id>
        <name>FuseSource Community Snapshot Repository</name>
        <url>http://repo.fusesource.com/nexus/content/groups/public-snapshots</url>
      </repository>
    </repositories>
    
    <dependencies>
      <dependency>
        <groupId>org.fusesource.leveldbjni</groupId>
        <artifactId>leveldbjni</artifactId>
        <version>1.0-SNAPSHOT</version>
      </dependency>
      <dependency>
        <groupId>org.fusesource.leveldbjni</groupId>
        <artifactId>leveldbjni</artifactId>
        <version>1.0-SNAPSHOT</version>
        <classifier>osx</classifier>
      </dependency>
    </dependencies>

## API Usage:

Recommended Package imports:

    import org.fusesource.leveldbjni.*;
    import static org.fusesource.leveldbjni.DB.*;
    import java.io.*;

Opening and closing the database.

    Options options = new Options();
    options.setCreateIfMissing(true);
    DB db = DB.open(options, new File("example"));
    try {
      // Use the db in here....
    } finally {
      // Make sure you delete the db to shutdown the 
      // database and avoid resource leaks.
      db.delete();
    }

Putting, Getting, and Deleting key/values.

    WriteOptions wo = new WriteOptions();
    ReadOptions ro = new ReadOptions();

    db.put(wo, bytes("Tampa"), bytes("rocks"));
    String value = asString(db.get(ro, bytes("Tampa")));
    db.delete(wo, bytes("Tampa"));

Performing Batch/Bulk/Atomic Updates.

    WriteBatch batch = new WriteBatch();
    try {
      batch.delete(bytes("Denver"));
      batch.put(bytes("Tampa"), bytes("green"));
      batch.put(bytes("London"), bytes("red"));

      WriteOptions wo = new WriteOptions();
      db.write(wo, batch);
    } finally {
      // Make sure you delete the batch to avoid resource leaks.
      batch.delete();
    }

Iterating key/values.

    ReadOptions ro = new ReadOptions();
    Iterator iterator = db.iterator(ro);
    try {
      for(iterator.seekToFirst(); iterator.isValid(); iterator.next()) {
        String key = asString(iterator.key());
        String value = asString(iterator.value());
        System.out.println(key+" = "+value);
      }
    } finally {
      // Make sure you delete the iterator to avoid resource leaks.
      iterator.delete();
    }

Working against a Snapshot view of the Database.

    ReadOptions ro = new ReadOptions();
    ro.setSnapshot(db.getSnapshot());
    try {
      
      // All read operations will now use the same 
      // consistent view of the data.
      ... = db.iterator(ro);
      ... = db.get(ro, bytes("Tampa"));

    } finally {
      // Make sure you release the snapshot to avoid resource leaks.
      db.releaseSnapshot(ro.getSnapshot());
      ro.setSnapshot(null);
    }

Using a custom Comparator.

    Options options = new Options();
    options.comparator(new Comparator(){
        int compare(byte[] key1, byte[] key2) {
            return new String(key1).compareTo(new String(key2));
        }
        String name() {
            return "simple";
        }
    });
    DB db = DB.open(options, new File("example"));

Disabling Compression

    Options options = new Options();
    options.compression(CompressionType.kNoCompression);
    DB db = DB.open(options, new File("example"));

Configuring the Cache
    
    Cache cache = new Cache(100 * 1048576);  // 100MB cache
    Options options = new Options();
    options.cache(cache);
    DB db = DB.open(options, new File("example"));
    ... use the db ...
    db.delete()
    cache.delete()

Getting approximate sizes.

    long[] sizes = db.getApproximateSizes(new Range(bytes("a"), bytes("k")), new Range(bytes("l"), bytes("z")));
    System.out.println("Size: "+sizes[0]+", "+sizes[1]);
    
Getting database status.

    String stats = db.getProperty("leveldb.stats");
    System.out.println(stats);
    