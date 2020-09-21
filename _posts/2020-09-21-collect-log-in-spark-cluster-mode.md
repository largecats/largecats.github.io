---
layout: post
title:  "Collecting Log in Spark Cluster Mode"
date:   2020-09-21
categories: work
tags: spark YARN Linux-shell
---
<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>
* content
{:toc}

# Motivation   

We want to take advantage of cluster mode in terms of resource while retaining the ability to 
1. access and store logs for recent as well as historical jobs conveniently, and
2. view log conveniently in real time.



# Background   

Spark has 2 deployment modes, client mode and cluster mode. The differences are where the driver program runs (and thus whose resource the driver program uses) and who is responsible for communication with YARN and requesting resource from YARN.

**Client mode**   

* driver program runs on driver server (i.e., the server that launches the spark-submit program) and uses its resource
* driver program is responsible for communication with YARN (it commands application master to request resource from YARN)


**Cluster mode**   

* driver program runs on the application master and uses the cluster resource
* application master is responsible for both communication with YARN and requesting resource from YARN

Client mode is suitable for cases (see [here](https://stackoverflow.com/questions/41124428/spark-yarn-cluster-vs-client-how-to-choose-which-one-to-use/41142747)) where
* jobs are interactive spark-shell, 
* jobs need to process local data on the driver server (since such data would not be readily available to the driver program in cluster mode).
  
Cluster mode is suitable for jobs that process large volume of data and require much resource for the driver program. This is an ideal deploy mode for batch ETL jobs, especially when the driver server is becoming the resource bottleneck. However in cluster mode, logs are not readily accessible as they are in client mode since they are generated to the stdout of different machines on the cluster, and in most cases, we only have YARN's log aggregation to rely on. The following sections explore a number of different workarounds and present a final solution that makes accessing and storing logs easy in cluster mode.

# Approaches   

## Redirect log to console in real time   

In cluster mode, logs are generated to the stdout stream of machines other than the server that submits the application. Our search did not turn up any method that can write logs to our chosen directory in real time or print it to console: 

* This post https://stackoverflow.com/questions/23058663/where-are-logs-in-spark-on-yarn/26082707#26082707 does not say can
* This post https://stackoverflow.com/questions/46725949/log4j-not-logging-in-spark-yarn-cluster-mode seems to want to do the same thing as we do, but no solutions are raised
* This post https://stackoverflow.com/questions/51014377/spark-driver-logs-on-edge-node-in-cluster-mode says cannot

## Collect aggregated log after application is finished   

Instead, we want to collect the aggregated logs to a designated directory using the `yarn logs -applicationId $applicationId` command after the spark application is finished. E.g., to collect log in HDFS:
```
yarn logs -applicationId $applicationId -log_files stdout -am 1 | hadoop fs -appendToFile - /user/xxx/log_--dates_2020-09-21.txt
```
or in driver server's local file system:
```
yarn logs -applicationId $applicationId -log_files stdout -am 1 |& tee -a /home/xxx/log_--dates_2020-09-21.txt
```
To view log in real-time, we could either use the `watch` and `tail` commands or rely on YARN's UI.

This approach means we need to consider two things:

1. applicationId: Where to get the applicationId that is needed to collect the aggregate logs from YARN, and
2. flow control: How to make sure that log collection is triggered only after the application is finished (either exiting normally or terminating upon error), so that the log is complete.

The applicationId is conveniently available via spark.sparkContext.applicationId in the .py script after creating a spark session, but the flow control is more conveniently done in shell script where we have a clear idea of when the spark-submit process terminates. We can put the log collection command in either the .py script or the shell script, with the understanding that either approach has trade-off.

### In .py script: Use customized sys.excepthook to trigger log collection upon any exception (X)   

In .py script, the applicationId can be easily accessed via spark.sparkContext.applicationId. But to put the log collection command in .py script, we need to make sure that any event that triggers program termination, be it normal termination or exception, is caught and redirected to log collection. E.g., we could wrap a try...except block around the main entry point of the driver script in the pipeline. Yet this does not capture errors in the definition of the pipeline itself, or our customized common modules that are imported by the driver script. Instead, we considered using a customized sys.excepthook to catch exceptions globally.

A customized excepthook is a function with 3 arguments `type, value, tback` and performs customized handling of the caught exception. In the example below, `except_hook`  returns a customized excepthook which, upon catching any exception, collects log using the applicationId supplied and calls the default handler `sys.__excepthook__()` to handle the exception.

```python
def collect_log(applicationId, logPath, logger):
    cmd = 'yarn logs -applicationId {application_id} -am 1 |& tee -a $logPath'.format(
            application_id=applicationId, logPath=logPath)
    logger.info('Collecting log to ' + logPath)
    logger.info(cmd)
    subprocess.call(cmd, shell=True)
    return True

def except_hook(applicationId, logPath, logger):
    def handler(type, value, tback):
        exc_info = sys.exc_info()
        # Display the *original* exception
        traceback.print_exception(*exc_info)
        del exc_info
        collect_log(applicationId, logPath, logger)
        sys.__excepthook__(type, value, tback)
    return handler

...

sys.excepthook = except_hook(applicationId, logPath, logger) # define customized excepthook
```
This approach, however, is flawed in that the definition of our customized excepthook requires a running spark session, which only exists in the config scripts of each data model, and not in the common modules. Other drawbacks include the need to carefully manage the order of imported modules, e.g., the customized excepthook should be defined before other objects in the driver script, or some errors may be missed and the log collection step wouldn't be triggered.

### In shell script: Parse client process' output to get applicationId and trigger log collection after spark-submit is finished   

Another approach we considered and ended up adopting is to trigger log collection in shell script after the spark-submit process is finished. This has the advantage of providing a clear indication when the spark-submit process terminates. The trade-off, though, is that instead of getting the applicationId conveniently from the spark session, we need to parse the spark-submit process' output and extract the applicationId.

A naive approach would be to print the applicationId from the spark session to pass it to the shell script. But the only way to do so is by printing the applicationId, and in cluster mode, this would go to the stdout of another machine in the cluster, which we cannot access from the driver server.

After some digging, we found that in cluster mode, the spark-submit command is launched by a client process, which starts on the driver server and exits as soon as it fulfills its responsibility of submitting the application to the cluster without waiting for the application to finish. The log of this client process contains the applicationId, and this log - because the client process is run by the driver server - can be printed to the driver server's console. In other words, this is the only place where the shell script can access the spark job's applicationId.

[pic TBD]

#### Implementation

**Print client process log to console**   

1. Add a log4j.properties file as follows. AS shown above, the applicationId in the client process' log is INFO level. So we need to set log4j.logger.Client  to INFO  level.
```
log4j.rootLogger=INFO, stdout, stderr
log4j.logger.Client = INFO, stdout, stderr
```
2. Pass the above log4j.properties file to the `SPARK_SUBMIT_OPTS` of the driver server that runs the client process. This gets the client process' log to be printed to the driver server's console.
```
SPARK_SUBMIT_OPTS="-Dlog4j.debug=true -Dlog4j.configuration=file:///home/xxx/config/log4j.properties" \
python spark_submit.py \
spark-submit --verbose \
--deploy-mode cluster \
--master yarn \
--name "My Spark Application $param" \
--conf spark.executor.memoryOverhead=5G \
--conf spark.yarn.maxAppAttempts=1 \
--num-executors 10 \
--py-files /home/xxx/lib.zip,/home/xxx/config.py \ # need to include all files, otherwise the driver program won't be able to find them from the cluster
/home/xxx/main.py $param
```

**Parse client process log to get applicationId**   

1. Launch the client process via `run`.
2. Set trap for the interruption signals SIGINT, SIGTERM to kill the application upon receiving these signals.
   1. Again, because spark applications in cluster mode run on the cluster, they will not receive any termination signal sent via the driver server. This means Ctrl-C (SIGINT), marking success or killing in Airflow (SIGTERM) would only kill the client process, but not the spark application itself. Instead, we need to explicitly invoke the yarn application -kill $applicationId  command upon receiving the termination signals.
3. Read the client process' log line by line until reaching the line containing the applicationId. Extract the applicationId.

```
#!/bin/bash

# constants
HDFS_DIR="/user/xxx"
DIR="/home/xxx"
NAME="my_spark_application"

# helper functions
source "$DIR/shell_functions.sh"

# directory to store logs
LOG_PATH="$DIR/logs/$NAME"
mkdir -p $LOG_PATH

run() {
    SPARK_SUBMIT_OPTS="-Dlog4j.debug=true -Dlog4j.configuration=file:///home/xxx/config/log4j.properties" \
    python spark_submit.py \
    spark-submit --verbose \
    --deploy-mode cluster \
    --master yarn \
    --name "My Spark Application $param" \
    --conf spark.executor.memoryOverhead=5G \
    --conf spark.yarn.maxAppAttempts=1 \
    --num-executors 10 \
    --py-files /home/xxx/lib.zip,/home/xxx/config.py \ # need to include all files, otherwise the driver program won't be able to find them from the cluster
    /home/xxx/main.py $param
}

# construct name of log file from parameters
param=${@}
logFileName=$NAME
for arg in ${param[@]}
    do
        if [[ ! $arg =~ "$HDFS_DIR" ]]
        then
            logFileName+="_$arg"
        else
            logFileName+="_${arg//\//-}" # change / to - in path so that the path can appear in the name of the log file
        fi
    done

APPLICATION_ID=""

# prepare to kill application via YARN upon receiving SIGINT (Ctrl-C), SIGTERM (Airflow) signals
trap 'keyboard_interrupt $APPLICATION_ID' SIGINT SIGTERM # use single quotes to delay variable expansion till when the function is called
trapExit=$?

# read output of run() line by line to find applicationId
while read -r line; do
    echo "$line"
    if [[ $line =~ "Submitting application application_" ]]
    then
        APPLICATION_ID=$(echo $line | grep -oP "application_[0-9_]*") # extract applicationId
        echo "Found applicationId: $APPLICATION_ID, continue running..."
    fi
done < <(run 2>&1)
get_application_status $APPLICATION_ID
sparkSubmitExit=$?

...

```

where `shell_functions.sh` include:

```

#!/bin/bash

keyboard_interrupt() {
    echo "Interrupted by keyboard."
    applicationId=$1
    if [ $applicationId != "" ]
    then
        yarn application -kill $applicationId
        return 1 # still need to collect log, cannot exit yet
    else
        echo "Application not yet submitted. Skipping kill via YARN..."
        exit_with_code 1 # no need to collect log, can directly exit
    fi
}

get_application_status() {
    applicationId=$1
    status=$(yarn application -status $applicationId 2>&1)
    echo "$status"
    if [[ "${status}" =~ "Final-State : SUCCEEDED" ]]
    then
        return 0
    else
        return 1
    fi
}

...

```

**Collect YARN aggregated logs to designated directory upon job completion**   

Use the extracted applicationId to collect logs to designated directory after the application is finished.

```
#!/bin/bash

...

# collect aggregated logs to designated directory
sleep 5
collect_log "$APPLICATION_ID" "$LOG_PATH/$logFileName.txt" 10 3
collectLogExit=$?

# exiting
finalExit=$(($trapExit+$sparkSubmitExit+$collectLogExit))
exit_with_code $finalExit
```

where `shell_functions.sh` include:

```

#!/bin/bash

...

collect_log_helper() {
    applicationId=$1
    logPath=$2

    echo "Collecting log..."
    output=$(yarn logs -applicationId $applicationId -log_files stdout stderr -am 1 2>&1)
    status=$(yarn application -status $applicationId 2>&1)
    if [[ "$output" =~ "Can not find" ]] || [[ "$output" =~ "Unable to get" ]] || [[ "$output" =~ "does not exist" ]] # log aggregation not ready yet
    then
        echo "$output"
        if [[ ! "$status" =~ "Log Aggregation Status : SUCCEEDED" ]]
        then
            echo "$status"
            if [[ "$status" =~ "Log Aggregation Status : NOT_START" ]] || [[ "$status" =~ "Log Aggregation Status : N/A" ]]
            then
                echo "Log aggregation not started. Skipping log collection..." # usually because log is not generated, e.g., application was killed before it started runnning
                return 0
            elif [[ "$status" =~ "Log Aggregation Status : DISABLED" ]]
            then
                echo "Log aggregation disabled. Skipping log collection..."
                return 0
            else
                echo "Log aggregation incomplete. Waiting for retry..."
                return 1
            fi
        else
            return 1
        fi
    else
        # echo "yarn logs -applicationId $applicationId -log_files stdout stderr -am 1 | hadoop fs -appendToFile - $logPath"
        # yarn logs -applicationId $applicationId -log_files stdout stderr -am 1 | hadoop fs -appendToFile - $logPath
        echo "yarn logs -applicationId $applicationId -log_files stdout stderr -am 1 |& tee -a $logPath"
        (yarn logs -applicationId $applicationId -log_files stdout stderr -am 1 |& tee -a $logPath) >/dev/null
        statuses=( "${PIPESTATUS[@]}" ) # copy PIPESTATUS to array statuses
        yarnCmdStatus=${statuses[$(( ${#statuses[@]} - 2 ))]}
        hadoopCmdStatus=${statuses[$(( ${#statuses[@]} - 1 ))]}
        if [ ${yarnCmdStatus} -eq 0 ] && [ ${hadoopCmdStatus} -eq 0 ]
        then
            return 0
        else
            return 1
        fi
    fi
}

collect_log() {
    applicationId=$1
    if [ $applicationId != "" ]
    then
        logPath=$2
        retryInterval=$3
        maxRetryNo=$4
        collect_log_helper $applicationId $logPath
        RET=$?
        while [[ ${RET} -ne 0 && ${maxRetryNo} -gt 0 ]]; do
            echo "Log collection failed, retrying in $retryInterval seconds..."
            sleep $retryInterval
            collect_log_helper $applicationId $logPath
            RET=$?
            maxRetryNo=$((maxRetryNo-1))
        done
        if [ ${maxRetryNo} -eq 0 ]
        then
            echo "Maximum number of retries reached. Aborting log collection..."
            return 1
        else
            return $RET
        fi
    else
        echo "No applicationId. Skipping log collection..." # technically this won't happen
        return 0
    fi
}


exit_with_code() {
    exitCode=$1
    echo "Exiting with code $exitCode."
    exit ${exitCode}
}
```

#### Workflow
[YARN's spark application statuses](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YarnCommands.html#application) are:

(pic - TBD)

[YARN's log aggregation statuses](http://hadoop.apache.org/docs/r3.1.0/hadoop-yarn/hadoop-yarn-api/apidocs/org/apache/hadoop/yarn/api/records/LogAggregationStatus.html) are:

(pic - TBD)

When the spark application terminates normally or upon exception, `collect_log` will be triggered to copy the logs aggregated by YARN to a designated directory.

What to do when the spark application is killed (either by Ctrl-C or via Airflow), on the other hand, requires extra handling.

If the job is killed before reaching SUBMITTED status
* `keyboard_interrupt` won't be invoked as the spark application is not yet launched;
* `collect_log` won't be triggered as no log is generated.

If the job is killed in or after SUBMITTED status but before reaching RUNNING status
* `keyboard_interrupt` will be invoked to kill the application before eventually exiting from the client process;
* `collect_log` won't be triggered as no log is generated.

If the job is killed after RUNNING status
* `keyboard_interrupt` will be invoked to kill the application before eventually exiting from the client process;
* `collect_log` will be triggered to collect the YARN aggregated logs after the application is killed.