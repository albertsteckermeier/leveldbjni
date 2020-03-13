# LevelDB JNI

## Description

LevelDB JNI gives you a Java interface to the 
[LevelDB](http://code.google.com/p/leveldb/) C++ library
which is a fast key-value storage library written at Google 
that provides an ordered mapping from string keys to string values.. 

# Getting the JAR

Just add the following jar to your java project:
[leveldbjni-all-1.8.jar](http://repo2.maven.org/maven2/org/fusesource/leveldbjni/leveldbjni-all/1.8/leveldbjni-all-1.8.jar)

## Using as a Maven Dependency

You just need to add the following dependency to your Maven POM:

    <dependencies>
      <dependency>
        <groupId>eu.cqse.leveldbjni</groupId>
        <artifactId>leveldbjni-all</artifactId>
        <version>1.9</version>
      </dependency>
    </dependencies>

By using the `leveldbjni-all` dependency, you get the OS specific native drivers for all supported platforms.

If you want to use only one or some but not all native drivers, then directly use the OS specific dependency instead of `leveldbjni-all`. For example to use Linux 64 bit, use this dependency:

    <dependencies>
      <dependency>
        <groupId>eu.cqse.leveldbjni</groupId>
        <artifactId>leveldbjni-linux64</artifactId>
        <version>1.9</version>
      </dependency>
    </dependencies>

If you have the leveljni native driver DLL/SO library already separately installed e.g. by a package manager (see [issue 90](https://github.com/fusesource/leveldbjni/issues/90)), then you could depend on the Java "launcher" without the JAR containing the OS specific native driver like this:

      <dependency>
        <groupId>eu.cqse.leveldbjni</groupId>
        <artifactId>leveldbjni</artifactId>
        <version>1.9</version>
      </dependency>

Lastly, another project unrelated to this project separately provides a (less mature) pure Java implementation of LevelDB, see [dain/leveldb](https://github.com/dain/leveldb).  Note that both that and this project share the same Maven artefact for the Level DB API interface (org.iq80.leveldb:leveldb-api).


## API Usage:

Recommended Package imports:

    import org.iq80.leveldb.*;
    import static org.fusesource.leveldbjni.JniDBFactory.*;
    import java.io.*;

Opening and closing the database.

    Options options = new Options();
    options.createIfMissing(true);
    DB db = factory.open(new File("example"), options);
    try {
      // Use the db in here....
    } finally {
      // Make sure you close the db to shutdown the 
      // database and avoid resource leaks.
      db.close();
    }

Putting, Getting, and Deleting key/values.

    db.put(bytes("Tampa"), bytes("rocks"));
    String value = asString(db.get(bytes("Tampa")));
    db.delete(bytes("Tampa"));

Performing Batch/Bulk/Atomic Updates.

    WriteBatch batch = db.createWriteBatch();
    try {
      batch.delete(bytes("Denver"));
      batch.put(bytes("Tampa"), bytes("green"));
      batch.put(bytes("London"), bytes("red"));

      db.write(batch);
    } finally {
      // Make sure you close the batch to avoid resource leaks.
      batch.close();
    }

Iterating key/values.

    DBIterator iterator = db.iterator();
    try {
      for(iterator.seekToFirst(); iterator.hasNext(); iterator.next()) {
        String key = asString(iterator.peekNext().getKey());
        String value = asString(iterator.peekNext().getValue());
        System.out.println(key+" = "+value);
      }
    } finally {
      // Make sure you close the iterator to avoid resource leaks.
      iterator.close();
    }

Working against a Snapshot view of the Database.

    ReadOptions ro = new ReadOptions();
    ro.snapshot(db.getSnapshot());
    try {
      
      // All read operations will now use the same 
      // consistent view of the data.
      ... = db.iterator(ro);
      ... = db.get(bytes("Tampa"), ro);

    } finally {
      // Make sure you close the snapshot to avoid resource leaks.
      ro.snapshot().close();
    }

Using a custom Comparator.

    DBComparator comparator = new DBComparator(){
        public int compare(byte[] key1, byte[] key2) {
            return new String(key1).compareTo(new String(key2));
        }
        public String name() {
            return "simple";
        }
        public byte[] findShortestSeparator(byte[] start, byte[] limit) {
            return start;
        }
        public byte[] findShortSuccessor(byte[] key) {
            return key;
        }
    };
    Options options = new Options();
    options.comparator(comparator);
    DB db = factory.open(new File("example"), options);
    
Disabling Compression

    Options options = new Options();
    options.compressionType(CompressionType.NONE);
    DB db = factory.open(new File("example"), options);

Configuring the Cache
    
    Options options = new Options();
    options.cacheSize(100 * 1048576); // 100MB cache
    DB db = factory.open(new File("example"), options);

Getting approximate sizes.

    long[] sizes = db.getApproximateSizes(new Range(bytes("a"), bytes("k")), new Range(bytes("k"), bytes("z")));
    System.out.println("Size: "+sizes[0]+", "+sizes[1]);
    
Getting database status.

    String stats = db.getProperty("leveldb.stats");
    System.out.println(stats);

Getting informational log messages.

    Logger logger = new Logger() {
      public void log(String message) {
        System.out.println(message);
      }
    };
    Options options = new Options();
    options.logger(logger);
    DB db = factory.open(new File("example"), options);

Destroying a database.
    
    Options options = new Options();
    factory.destroy(new File("example"), options);

Repairing a database.
    
    Options options = new Options();
    factory.repair(new File("example"), options);

Using a memory pool to make native memory allocations more efficient:

    JniDBFactory.pushMemoryPool(1024 * 512);
    try {
        // .. work with the DB in here, 
    } finally {
        JniDBFactory.popMemoryPool();
    }
    
## Building

See also [releasing.md](releasing.md):

Building all the relevant Jars of this Maven project requires multiple steps being performed across different operation systems. The general process is as follows.

1. Build and install main Jars as well as Java JNI C sources on Linux. This also already builds the Linux binary Jar.
2. Zip and move repo as well as artifacts from local Maven to Windows machine and build only the Windows binaries there.
3. Zip and move repo as well as artifacts from local Maven on the Windows machine to a macOS machine and build the macOS binaries there.
4. Build the uber Jar containing all binaries as well as Java class needed to use the LevelDB JNI across Linux, Windows and macOS.

### General

In order to make the somewhat outdated build work there exist some forks of the required repositories. Please checkout all three. For the current build the `_leveldbjni` branches have been used.

* [Snappy](https://github.com/albertsteckermeier/snappy)
    * At the time of writing Snappy 1.1.8 was used
    * Apply [this fix commit](https://github.com/albertsteckermeier/snappy/commit/0c5e3b2d82650bfd1eeabd6afb98d2d8a30226ba) as well to make the LevelDB JNI build work
* [LevelDB](https://github.com/albertsteckermeier/leveldb)
    * At the time of writing LevelDB 1.22 was used
    * Apply [this fix commit](https://github.com/albertsteckermeier/leveldb/commit/ac1b5a869eca43df7f3277e8b5118d14ed54b9af) as well to make the LevelDB JNI build work
* [LevelDB JNI](https://github.com/albertsteckermeier/leveldbjni) 
    * Latest master contains some modifications to make the builds work
    
Also **make sure to update the version numbers** in all the Maven POM XML files to avoid Jar clashes before starting to build a new version.

### Linux

#### Prerequisites

* Maven 3
* GNU Autoconf
* GNU Libtool
* CMake

#### Build

1. Clone all the repositories locally.
1. Checkout the `_leveldbjni` branches or apply the fixes to a newer version.
1. Build Snappy
    1. `cd` to the Snappy repository directory
    1. Build with: `cmake . && make`
    1. Export environment variable for LevelDB JNI build: `export SNAPPY_HOME=$(pwd)`
1. Build LevelDB
    1. `cd` to the LevelDB repository directory
    1. Build with: `cmake -DCMAKE_BUILD_TYPE=Release . && cmake --build .`
    1. Export environment variable for LevelDB JNI build: `export LEVELDB_HOME=$(pwd)`
1. Build LevelDB JNI
    1. `cd` to the LevelDB JNI repository directory
    1. Build with: `mvn clean install -P download -P linux64`
1. After this process you should locally have Jars for `leveldbjni`, `leveldbjni-linux64` and `leveldbjni-project` in your local Maven repository at `~/.m2/eu/cqse/leveldbjni`.
1. Zip the LevelDB JNI repository as well as the local Maven repository contents at `~/.m2/eu/cqse/leveldbjni` and move both to your Windows machine.

### Windows

#### Prerequisites

* Maven 3
* Git Bash
* CMake
* Visual Studio 2019

#### Build

1. Clone the Snappy and LevelDB repositories locally.
1. Checkout the `_leveldbjni` branches or apply the fixes to a newer version.
1. Build Snappy
    1. Set system environment variable `SNAPPY_HOME` to the repository directory
    1. `cd` to the Snappy repository directory
    1. Create Visual Studio 2019 solution: `cmake -G "Visual Studio 16 2019" -A x64 .`
    1. Open `Snappy.sln` in Visual Studio
    1. Select solution configuration `Release` and solution platform to `x64`
    1. Right click the `snappy` solution and set _Properties > C/C++ > Code Generation > Runtime_ Library to `Multi-threaded (/MT)` instead of `Multi-threaded DLL (/MT)`
    1. Right click the `snappy` solution and build/rebuild it.
    1. You should now have a Snappy Lib file in the `Release` folder
1. Build LevelDB
    1. Set system environment variable `LEVELDB_HOME` to the repository directory
    1. `cd` to the LevelDB repository directory
    1. Create Visual Studio 2019 solution: `cmake -G "Visual Studio 16 2019" -A x64 .`
    1. Open `leveldb.sln` in Visual Studio
    1. Select solution configuration `Release` and solution platform to `x64`
    1. Right click the `leveldb` solution and set _Properties > C/C++ > Code Generation > Runtime_ Library to `Multi-threaded (/MT)` instead of `Multi-threaded DLL (/MT)`
    1. Right click the `leveldb` solution and build/rebuild it.
    1. You should now have a LevelDB Lib file in the `Release` folder
1. Build only LevelDB JNI binary for Windows
    1. Extract the copied LevelDB JNI repository.
    1. Extract the Maven repository contents to the corresponding Windows local Maven repository at `C:\Users\<Username>\.m2\eu\cqse\leveldbjni`
    1. `cd` to the extracted LevelDB JNI repository directory
    1. Build only the Windows Jar with: `cd leveldbjni-win64 && mvn clean install`
    1. You should have a folder `target/generated-sources/hawtjni/native-package` which contains a `vs2010.vcxproj` file.
    1. Open the `vs2010.vcxproj` in Visual Studio 2019 and Upgrade to the latest SDK.
    1. Make sure to have your `JAVA_HOME` environment variable set
    1. Select solution configuration `Release` and solution platform to `x64`
    1. Right click the `leveldb` solution and set _Properties > C/C++ > Code Generation > Runtime_ Library to `Multi-threaded (/MT)` instead of `Multi-threaded DLL (/MT)`
    1. Right click the `leveldbjni` solution and build/rebuild it.
    1. You should now have a LevelDB JNI DLL in the `Release` folder
    1. Put that DLL into the Jar `leveldbjni-win64/target/leveldbjni-win64-<Version>.jar` in subfolder `META_INF/native/windows64` within the Jar file.
    1. Override the locally installed copy in the local Maven repository at `C:\Users\<Username>\.m2\eu\cqse\leveldbjni\leveldbjni-win64` with the Jar file containing the DLL-
1. Zip the LevelDB JNI repository as well as the local Maven repository contents at `C:\Users\<Username>\.m2\eu\cqse\leveldbjni` and move both to your macOS machine.

### macOS

These steps have been performed on macOS Catalina. You may have to add to the `OSX_VERSIONS` for newer versions of macOS if they include newer SDKs.

#### Prerequisites

* Maven 3
* GNU Autoconf
* GNU Libtool
* CMake

#### Build

1. Clone the Snappy and LevelDB repositories locally.
1. Checkout the `_leveldbjni` branches or apply the fixes to a newer version.
1. Build Snappy
    1. `cd` to the Snappy repository directory
    1. Build with: `cmake . && make`
    1. Export environment variable for LevelDB JNI build: `export SNAPPY_HOME=$(pwd)`
1. Build LevelDB
    1. `cd` to the LevelDB repository directory
    1. Build with: `cmake -DCMAKE_BUILD_TYPE=Release . && cmake --build .`
    1. Export environment variable for LevelDB JNI build: `export LEVELDB_HOME=$(pwd)`
1. Build only LevelDB JNI binray for macOS
    1. Extract the copied LevelDB JNI repository.
    1. Extract the Maven repository contents to the corresponding Windows local Maven repository at `~/.m2/eu/cqse/leveldbjni`
    1. `cd` to the LevelDB JNI repository directory
    1. Build with: `cd leveldbjni-osx && mvn clean install`
1. After this process you should locally have Jars for `leveldbjni`, `leveldbjni-linux64`, `leveldbjni-win64`, `leveldbjni-osx` and `leveldbjni-project` in your local Maven repository at `~/.m2/eu/cqse/leveldbjni`.

### Final steps

These steps can be performed on any OS.

1. Make sure to move all the Jars for `leveldbjni`, `leveldbjni-linux64`, `leveldbjni-win64`, `leveldbjni-osx` and `leveldbjni-project` in your local Maven repository.
1. Make sure to use the LevelDB JNI repository that contains all the Jars. In our example build steps outlined above this would be the repository present on macOS.
1. Build the uber Jar containing everything
    1. `cd` to the LevelDB JNI repository
    1. Build the uber Jar: `cd leveldbjni-all && mvn clean install -P release`
1. You should now have all Jars for Jars for `leveldbjni`, `leveldbjni-linux64`, `leveldbjni-win64`, `leveldbjni-osx`, `leveldbjni-all` and `leveldbjni-project` in your local Maven repository
1. Publish the contents of your local repository at `~/.m2/eu/cqse/leveldbjni` to the artifacts SVN.
