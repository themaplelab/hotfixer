# HOTFIXER
  * This repo contains HOTFIXER, a tool for hotfixing Crypto API misuses in Java applications. HOTFIXER is built of many different software components, all of which are listed below. HOTFIXER was developed and tested in [Docker](https://www.docker.com/), however the steps should remain roughly the same on Ubuntu (18.04) if one should choose.
  * most of these steps could be throw into a script and you could walk away while it does its thing. but if you do that, stick around for the `apt-get`s, actually, probably do those first so you can hit `y` (yes you want to take up space/install ect.)
  * the setup time overall will vary depending on compute power ect of your machine, but this overall does take a while, expect at least 1/2 an hour - 1 hour of time since there are several cloning and building steps.


# SETUP
## JVM BUILD
* to build the JVM and setup, follow these intructions in order. The JVM must be built before all other components.
1) get a Docker container up and running for this:
    ```
    wget https://raw.githubusercontent.com/eclipse/openj9/master/buildenv/docker/mkdocker.sh

    chmod +x mkdocker.sh

    ./mkdocker.sh --tag=hotfixerdemo --dist=ubuntu --version=18.04 --build

    docker run -it hotfixerdemo
    ```
2) attach to that docker container
* either docker run with the interactive option will have attached you, or you can re-attach to your container later after detaching, using `docker attach CONTAINER`
* `docker ps` and `docker ps -a` will show you more information about which containers you have in your environment.

3) get and build the JVM (3 components: [extended version of OpenJDK8](https://github.com/ibmruntimes/openj9-openjdk-jdk8), [Eclipse OpenJ9 JVM](eclipse.org/openj9/), [Eclipse OMR](https://github.com/eclipse/omr))
    ```
    cd root
    
    apt update

    apt-get install emacs

    git clone https://github.com/ibmruntimes/openj9-openjdk-jdk8.git

    cd openj9-openjdk-jdk8

    git  checkout -b hotfixerbuild 6a92a55d726cc48b18118b79dcb67d34e45dd8f7

    ./get_source.sh -openj9-repo=https://github.com/themaplelab/openj9.git -openj9-branch=experiment-version-latest -omr-repo=https://github.com/themaplelab/openj9-omr.git -omr-branch=experiment-version-latest
    ```
4) get protobuf, which is currently required for the jvm build
    ```
    cd  ~/

    wget https://github.com/protocolbuffers/protobuf/releases/download/v3.7.1/protobuf-cpp-3.7.1.tar.gz

    tar -xvzf protobuf-cpp-3.7.1.tar.gz

    cd protobuf-3.7.1

    ./configure --disable-shared --with-pic && make && make install && ldconfig

    cd ..
    ``` 

5) setup protobufs
    ```
    cd openj9-openjdk-jdk8/openj9/runtime/compiler/tcp/

    ./compileproto.sh

    cd ~/

    rm -rf /protobuf-3.7.1 && rm -rf /protobuf-cpp-3.7.1.tar.gz
    ```

6) ready to configure the build, then build the JVM
    ```
    cd openj9-openjdk-jdk8

    bash configure --with-freemarker-jar=/root/freemarker.jar

    make all
    ```

7) confirm JVM build was successful, the output of `/root/openj9-openjdk-jdk8/build/linux-x86_64-normal-server-release/images/j2sdk-image/bin/java -version` should look similar to this (after aliasing step we can do `java -version` but should check first that it worked before aliasing otherwise we might change a required bootJVM path). The timestamp related portions will be different but version type portions should look identical.
    ```
    openjdk version "1.8.0_252-internal"
    OpenJDK Runtime Environment (build 1.8.0_252-internal-_2020_11_23_22_29-b00)
    Eclipse OpenJ9 VM (build experiment-version-latest-526b917f5, JRE 1.8.0 Linux amd64-64-Bit Compressed References 20201124_000000 (JIT enabled, AOT enabled)
    OpenJ9   - 526b917f5
    OMR      - 296fcc8c3
    JCL      - 6a92a55d72 based on jdk8u252-b09)
    ```
### setup all correct aliases and env vars for our JVM build
* You may put them in bashrc, if you want, but for some reason even after I source the bashrc, my subshells only see the java alias but not the javac alias which is weird. At least with the snippet below you can easily copy the full javac path for future use, although that is a terrible way to have to invoke javac...
* just note, if you dont put this in bashrc or setup config, when the docker exits, the paths will need resetting
    ```
    export JAVA_HOME=/root/openj9-openjdk-jdk8/build/linux-x86_64-normal-server-release/images/j2sdk-image

    export LD_LIBRARY_PATH=/root/openj9-openjdk-jdk8/build/linux-x86_64-normal-server-release/images/j2sdk-image/jre/lib/amd64/compressedrefs:/root\
    /openj9-openjdk-jdk8/build/linux-x86_64-normal-server-release/images/j2sdk-image/jre/lib/amd64:/usr/lib64:/usr/lib:/usr/local/lib

    alias java='/root/openj9-openjdk-jdk8/build/linux-x86_64-normal-server-release/images/j2sdk-image/bin/java'

    alias javac='/root/openj9-openjdk-jdk8/build/linux-x86_64-normal-server-release/images/j2sdk-image/bin/javac'

    alias jar="/root/openj9-openjdk-jdk8/build/linux-x86_64-normal-server-release/images/j2sdk-image/bin/jar"

    export PATH=/root/openj9-openjdk-jdk8/build/linux-x86_64-normal-server-release/images/j2sdk-image/jre/bin/:$PATH
    ```

## Maven Environment Setup
* Have tested with both 3.6 and 3.5.2, but 3.5.2 was used during experiements. No forseen differences should occur.
    ```
    apt install maven

    mvn -v
    ```

## [Soot](http://soot-oss.github.io/soot/) Build
* First build the customized Soot, then add that Soot build to our local mvn repo, so that it is used by ssDiffTool and CogniCrypt
    ```
    cd ~/

    git clone https://github.com/themaplelab/soot.git -b classreaderExtLatest

    cd soot

    mvn -e clean compile assembly:single -DskipTests

    cd ..

    rm -rf /root/.m2/repository/soot

    mvn install:install-file -Dfile=/root/soot/target/sootclasses-trunk-jar-with-dependencies.jar -DgroupId=soot.local -DartifactId=localsoot -Dversion=4.5 -Dpackaging=jar
    ```

## build the Patch Adapter
* ssDiffTool is our patch adapter implementation. It uses our custom version of Soot.
    ```
    cd ~/

    git clone https://github.com/themaplelab/ssDiffTool.git -b finalExpVersion

    cd ssDiffTool

    mvn clean compile assembly:single
    ```

## [CogniCrypt](https://github.com/CROSSINGTUD/CryptoAnalysis) Build
* Next build our custom version of CogniCrypt.
### First Manage a CogniCrypt Dependency
* CogniCrypt uses CrySL, which includes some CrySL reader funcitionality ect. We used during version 2.0.0 of that during our experiments and want to make sure we use same version as then, otherwise default maven setup will fetch a later version.
    ```
    mkdir /root/.m2/repository/de/darmstadt/ /root/.m2/repository/de/darmstadt/tu/ /root/.m2/repository/de/darmstadt/tu/crossing/ /root/.m2/repository/de/darmstadt/tu/crossing/CrySL/ /root/.m2/repository/de/darmstadt/tu/crossing/CrySL/de.darmstadt.tu.crossing.CrySL/ /root/.m2/repository/de/darmstadt/tu/crossing/CrySL/de.darmstadt.tu.crossing.CrySL/2.0.0-SNAPSHOT/

    cp /root/ssDiffTool/agent/de.darmstadt.tu.crossing.CrySL-2.0.0-SNAPSHOT.jar /root/.m2/repository/de/darmstadt/tu/crossing/CrySL/de.darmstadt.tu.crossing.CrySL/2.0.0-SNAPSHOT/
    ```

### Now ready to build our custom CogniCrypt version
```
cd ~/

git clone https://github.com/themaplelab/CryptoAnalysis.git -b kristen-CogniTCPlatest

cd CryptoAnalysis/CryptoAnalysis

mvn package -e -X -DskipTests=true
```

## build the Java Agent 
* HOTFIXER uses this Java Agent to handle receiving the hotfix, and to do the class redefinition. This Java Agent uses the [Java Instrumentation API](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/Instrumentation.html).
    ```
    cd /root/ssDiffTool/agent

    ./make.sh

    #or just type `. make.sh` if the aliasing of javac didnt work for some reason in the bashrc or above aliasing.
    cd ~/

    mv /root/ssDiffTool/agent/agent.jar .
    ```

# HOW TO RUN
## CryptoGuardBench
```
# get the data
cd ~/
git clone https://github.com/themaplelab/cryptoapi-bench.git

# now we move some run scripts around 
cp /root/ssDiffTool/runScripts/allpatches.out .
cp /root/ssDiffTool/runScripts/runCogniServerBasic.sh /root/CryptoAnalysis/
cp /root/ssDiffTool/runScripts/runALLBasic.sh .
cp /root/ssDiffTool/runScripts/runHotFixBasic.sh .
cp /root/ssDiffTool/runScripts/allpatches.out /root/CryptoAnalysis/
cp /root/ssDiffTool/runScripts/org.cryptoapi.bench.staticsalts.StaticSaltsABHCase1.originalclasses.out /root/CryptoAnalysis/

# get gradle
apt-get install gradle

# then we run
./runALLBasic.sh
```
* some of the output is then sent to stdout, but all of it is also recorded in `/root/basicBmkResults/misuseX` folders
* quick guide to how the output is sorted, since HOTFIXER has many components:
    1) JIT/JVM output located in *TEST.txt files
    2) CogniServer/CogniCrypt output located in *ALL.txt files
    3) summary of both scraped into *REPORT.txt files
* the run script here is currently set to run 1 sample. It can be modified using the following steps:
  1) to run additional/alternative examples, modify the `repos` and `classes` sets in `runALLBasic.sh`
    * for example to run misuse2 we would check the contents of `/root/cryptoapi-bench/patch/misuse2/classes.txt ` to see what class to add to the classes set.
  2) add a file named [FQN-Class-for-misuse].originalclasses.out with the contents: `FQN-Class-for-misuse` to `/root/CryptoAnalysis/`. this tells the patch adapter what class set to look at for the class that is currently being hotfixed, in the event that patching ClassA requires also adjusting ClassB and ClassC

## WickertBench: ha-bridge example
```
#untar the example zip in ssDiffTool/runScripts

cd /root/ssDiffTool/runScripts
tar xvzf haexample.tar.gz
mv ha-bridge  /root/

cp /root/ssDiffTool/runScripts/CryptoAnalysis/runCogniServerAll.sh /root/CryptoAnalysis/
cp /root/ssDiffTool/runScripts/runALL.sh  /root/
cp /root/ssDiffTool/runScripts/runHotFixALL.sh /root/
cp /root/ssDiffTool/runScripts/allpatchesMU.out /root/CryptoAnalysis/
cp /root/ssDiffTool/runScripts/com.bwssystems.HABridge.BridgeSecurity.originalclasses.out /root/CryptoAnalysis/
cd /root/

# then run
./runALL.sh
```
* output for this run went into `/root/ha-bridgeHotFixResults/ `

### Extra running details
* look for "REDEFINITION PASSED" in the stdout and thats how you know at a glance that all went well !
* each test may take between 2-4 minutes to run since the scripts have some padding sleeps for the output scraping.
* the setup does not gracefully check for ports already in use. it is advised that you check running processes after each run to assure that no processes are left hanging which would impact future runs. Similarly, this cannot be parallelized at this time, without a more modular design for providing port numbers.
