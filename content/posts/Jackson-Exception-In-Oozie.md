---
title: "Jackson Exception in Oozie"
date: 2018-07-09T14:35:49-04:00
draft: false
---

## Jackson Exception in Oozie
When using Spark2 within an oozie workflow, I've noticed that out of the box Hortonworks provides conflicting jars with Jackson, which leads to an exception when you launch a Spark2 job.  The exception usually looks something like this:

{{< highlight text >}}
2018-07-06 14:45:53,763 [Thread-43] ERROR org.apache.spark.sql.execution.datasources.FileFormatWriter  - Aborting job null.
java.lang.NoSuchMethodError: com.fasterxml.jackson.databind.JavaType.isReferenceType()Z
        at com.fasterxml.jackson.databind.ser.BasicSerializerFactory.findSerializerByLookup(BasicSerializerFactory.java:302)
        at com.fasterxml.jackson.databind.ser.BeanSerializerFactory._createSerializer2(BeanSerializerFactory.java:218)
        at com.fasterxml.jackson.databind.ser.BeanSerializerFactory.createSerializer(BeanSerializerFactory.java:153)
        at com.fasterxml.jackson.databind.SerializerProvider._createUntypedSerializer(SerializerProvider.java:1203)
        at com.fasterxml.jackson.databind.SerializerProvider._createAndCacheUntypedSerializer(SerializerProvider.java:1157)
        at com.fasterxml.jackson.databind.SerializerProvider.findValueSerializer(SerializerProvider.java:481)
        at com.fasterxml.jackson.databind.SerializerProvider.findTypedValueSerializer(SerializerProvider.java:679)
        at com.fasterxml.jackson.databind.ser.DefaultSerializerProvider.serializeValue(DefaultSerializerProvider.java:107)
        at com.fasterxml.jackson.databind.ObjectMapper._configAndWriteValue(ObjectMapper.java:3559)
        at com.fasterxml.jackson.databind.ObjectMapper.writeValueAsString(ObjectMapper.java:2927)
        at org.apache.spark.rdd.RDDOperationScope.toJson(RDDOperationScope.scala:52)
        at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:142)
        at org.apache.spark.sql.execution.SparkPlan.executeQuery(SparkPlan.scala:152)
        at org.apache.spark.sql.execution.SparkPlan.execute(SparkPlan.scala:127)
        at org.apache.spark.sql.execution.datasources.FileFormatWriter$.write(FileFormatWriter.scala:180)
{{< / highlight >}}

In order to resolve it, I remove the older version of all jackson libraries:
{{< highlight text >}}
hdfs dfs -rm /user/oozie/share/lib/lib_<ts>/oozie/jackson* 
{{< / highlight >}}

and then update the oozie sharelib:
{{< highlight text >}}
oozie admin -sharelibupdate
{{< / highlight >}}
