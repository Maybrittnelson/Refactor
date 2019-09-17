> 创建表时需要指定数据切分格式，会用到ROW FORMAT关键字。

> ```sql
> row_format
>   ``: DELIMITED [FIELDS TERMINATED BY ``char` `[ESCAPED BY ``char``]] [COLLECTION ITEMS TERMINATED BY ``char``]
>         ``[MAP KEYS TERMINATED BY ``char``] [LINES TERMINATED BY ``char``]
>         ``[NULL DEFINED AS ``char``]   -- (Note: Available in Hive ``0.13` `and later)
>   ``| SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]
> ```
>
> DELIMITED（划定）:
>
> * FILEDS
> * COLLECTION
> * MAP
> * LINES
> * NULL
>
> 
>
> SERDE：
>
> What is a SerDe?
>
> - SerDe is a short name for "Serializer and Deserializer."
> - Hive uses SerDe (and FileFormat) to read and write table rows.
> - HDFS files --> InputFileFormat --> <key, value> --> Deserializer --> Row object
> - Row object --> Serializer --> <key, value> --> OutputFileFormat --> HDFS files

