---
title: "Building Oozie Workflows"
date: 2018-06-30T14:19:06-04:00
draft: false
---

## Building Oozie Workflows
Over here at Tufts there's an ongoing effort to convert legacy ETL processes over to our new Hadoop cluster.
At the cornerstone of such an effort lies the workflow scheduling tool [Oozie](https://oozie.apache.org/).  Oozie handles both the generation of DAG based workflows, as well as the scheduling of said workflows through markup described files.  Oozie can handle several different types of actions, whether they be executing scripts(bash/pig/hive/spark), running java programs, or performing actions within HDFS.

## Workflows
A simplistic Oozie workflow is described by a workflow.xml file that sits in the root level directory of your job, as well as a job.properties file.  Think of the job.properties file as a series of variables that get used within your workflow.  For this example we'll start building out a simple workflow that executes a hive script, and then runs a spark2 job.  We'll start this out by creating a directory to store our jobs and define our job.properties file:

{{< highlight shell >}}
mkdir oozie_workflow/
cd oozie_workflow && mkdir lib
vim job.properties
{{< / highlight >}}

#### job.properties
{{< highlight text >}}
nameNode=hdfs://<namenodeHostname>:8020
jobTracker=<jobtrackerHostname>:8050
resourceManager=<resourceManager>:8050
queueName=<yarnQueueName>
jdbcURL=jdbc:hive2://<hiveserver2_url>:10000/default
jdbcPrincipal=hive/_HOST@EXAMPLE.com
hcatPrincipal=hive/_HOST@EXAMPLE.com
user.name=testuser
oozie.use.system.libpath=true
oozie.action.sharelib.for.spark=spark2
oozie.wf.application.path=${nameNode}/user/test/example
{{< / highlight >}}

So that was a lot, so lets break down what every property is going to be used for.  All hive actions in Oozie are executed via an underlying beeline command, so we have to declare what JDBC uri we're going to be using, which yarn queue, and the hive kerberos principal we'll be using(assuming you have hive.server2.enable.doAs set to false set to false).  We declare our nameNode, jobTracker, and resourceManager to facilitate connections between beeline and our spark client and the hadoop cluster.  We're going to tell oozie to use the host systems libpath, and I'm declaring which version of spark that we want to use.  Setting up that libpath is out of the scope for this tutorial, but you can get the gist from [this hortonworks document](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.0/bk_spark-component-guide/content/ch_oozie-spark-action.html#spark-config-oozie-spark2).  The oozie.wf.aplication.path is set to tell oozie where in HDFS our underlying workflow exists.  So now that that's out of the way, lets create our workflow.xml file that'll declare what oozie should actually be DOING.

#### workflow.xml
{{< highlight xml >}}
<?xml version="1.0" encoding="UTF-8"?>
<workflow-app xmlns="uri:oozie:workflow:0.5" name="SampleWorkflow">
  <credentials>
    <credential name="hs2-creds" type="hive2">
      <property>
        <name>hive2.server.principal</name>
        <value>${jdbcPrincipal}</value>
      </property>
      <property>
        <name>hive2.jdbc.url</name>
        <value>${jdbcURL}</value>
      </property>
    </credential>
    <credential name="hcat-creds" type="hcat">
      <property>
        <name>hcat.metastore.uri</name>
        <!--this property is stored in hive.metastore.uris within your hive config.  
        It'll have the prefix thrift:// and is usually on port 9083 -->
        <value></value>
      </property>
      <property>
        <name>hcat.metastore.principal</name>
        <value>${hcatPrincipal}</value>
      </property>
    </credential>
</credentials>
<start to="hive2-1"/>
<action name="hive2-1" cred="hs2-creds">
    <hive2 xmlns="uri:oozie:hive2-action:0.1">
        <job-tracker>${jobTracker}</job-tracker>
        <name-node>${nameNode}</name-node>
        <!-- this is your hive-site.xml file thats stored in /usr/hdp/current/spark-client/conf -->
        <job-xml>hive-site.xml</job-xml>
        <configuration>
            <property>
                <name>mapred.job.queue.name</name>
                <value>${queueName}</value>
            </property>
        </configuration>
        <jdbc-url>${jdbcURL}</jdbc-url>
        <script>hivescript.hql</script>
      </hive2>
      <ok to="pyspark_script"/>
      <error to="end"/>
  </action>
  <action name="pyspark_script" cred="hcat-creds">
    <spark xmlns="uri:oozie:spark-action:0.1">
      <job-tracker>${jobTracker}</job-tracker>
      <name-node>${nameNode}</name-node>
      <master>yarn</master>
      <name>spark_test</name>
      <jar>pyspark_script.py</jar>
      <!-- if you usually pass additional properties to spark-submit you can uncomment this and provide them.  Its not needed though
      <spark-opts></spark-opts>-->
    </spark>
    <ok to="end"/>
    <error to="end"/>
  </action>
  <end name="end"/>
</workflow-app>
{{< / highlight >}}

If you use Kerberos (I do), you'll need to declare which principal HCAT should be using, as this is what hive will be authenticating through.

Actually scheduling this workflow is easy enough if you're familiar with cron.  Something important to note however, is that oozie will run historical jobs if your start time takes place in the past.  So if I were to put a timestamp such as 2018-07-08T10:00Z and the current date is 2018-07-11, then this workflow will immediately be executed 3 times. 

#### Coordinator.xml
{{< highlight xml >}}
<coordinator-app name="etl"
frequency="0 7 * * *"
start="${jobStart}" end="${jobEnd}" timezone="UTC"
xmlns="uri:oozie:coordinator:0.2">
    <action>
        <workflow>
            <app-path>${appPath}</app-path>
            <configuration>
                <property>
                    <name>jobTracker</name>
                    <value>${jobTracker}</value>
                </property>
                <property>
                    <name>nameNode</name>
                    <value>${nameNode}</value>
                </property>
                <property>
                    <name>queueName</name>
                    <value>${queueName}</value>
                </property>
            </configuration>
        </workflow>
    </action>
</coordinator-app>
{{< / highlight >}}

#### deploy.sh
{{< highlight shell >}}
hdfs dfs -copyFromLocal -f * /user/test/workflow && oozie job -config job.properties -run
{{< / highlight >}}
