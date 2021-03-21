# 【Spark】spark 如何获取 JDBC 的 Schema

Spark 读取 JDBC 数据的代码相当之简单, 但是在 code 的时候不禁有个疑问, Spark 如何知道 JDBC 中某个表的 Schema 的 ?

其本质是将 JDBC 表数据结构映射到 Spark DataFrame 的数据结构

```scala
object LoadFromJDBC {
	
	def main(args: Array[String]): Unit = {
		val sparkSession: SparkSession = SparkSession.builder()
				.appName("Jdbc reader")
				.master("local[*]")
				.getOrCreate()		
		val df: DataFrame = sparkSession.read.format("jdbc")
				.option("url", "jdbc:mysql://127.0.0.1:3306/employees")
				.option("driver", "com.mysql.jdbc.Driver")
				.option("user", "root")
				.option("password", "1234")
				.option("dbtable", "employees")
				.load()	
		df.show()
		sparkSession.close()
	}
}
```

我们从 load() 处一步步 debug 进去。

## 1. DataFrameReader#load()

DataFrameReader#load() 中关键的代码之一就是 

`val cls = DataSource.lookupDataSource(source, sparkSession.sessionState.conf)`

### 1.1 DataSource#lookupDataSource()

```scala
val provider1 = backwardCompatibilityMap.getOrElse(provider, provider) match {
  case name if name.equalsIgnoreCase("orc") &&
      conf.getConf(SQLConf.ORC_IMPLEMENTATION) == "native" =>
    classOf[OrcFileFormat].getCanonicalName
  case name if name.equalsIgnoreCase("orc") &&
      conf.getConf(SQLConf.ORC_IMPLEMENTATION) == "hive" =>
    "org.apache.spark.sql.hive.orc.OrcFileFormat"
  case name => name
}
```

> 这里的 backwardCompatibilityMap 集合中封装了 Spark 预定义的一些 DataSource。
>
> ```java
> // backwardCompatibilityMap 共 19 个预定义的 DataSource
> Map(org.apache.spark.sql.hive.orc.DefaultSource -> org.apache.spark.sql.hive.orc.OrcFileFormat, org.apache.spark.sql.execution.datasources.json -> org.apache.spark.sql.execution.datasources.json.JsonFileFormat, org.apache.spark.sql.execution.datasources.json.DefaultSource -> org.apache.spark.sql.execution.datasources.json.JsonFileFormat, org.apache.spark.ml.source.libsvm.DefaultSource -> org.apache.spark.ml.source.libsvm.LibSVMFileFormat, org.apache.spark.ml.source.libsvm -> org.apache.spark.ml.source.libsvm.LibSVMFileFormat, org.apache.spark.sql.execution.datasources.orc.DefaultSource -> org.apache.spark.sql.execution.datasources.orc.OrcFileFormat, org.apache.spark.sql.jdbc.DefaultSource -> org.apache.spark.sql.execution.datasources.jdbc.JdbcRelationProvider, org.apache.spark.sql.json.DefaultSource -> org.apache.spark.sql.execution.datasources.json.JsonFileFormat, org.apache.spark.sql.json -> org.apache.spark.sql.execution.datasources.json.JsonFileFormat, org.apache.spark.sql.execution.datasources.jdbc -> org.apache.spark.sql.execution.datasources.jdbc.JdbcRelationProvider, org.apache.spark.sql.parquet -> org.apache.spark.sql.execution.datasources.parquet.ParquetFileFormat, org.apache.spark.sql.parquet.DefaultSource -> org.apache.spark.sql.execution.datasources.parquet.ParquetFileFormat, org.apache.spark.sql.execution.datasources.parquet.DefaultSource -> org.apache.spark.sql.execution.datasources.parquet.ParquetFileFormat, com.databricks.spark.csv -> org.apache.spark.sql.execution.datasources.csv.CSVFileFormat, org.apache.spark.sql.hive.orc -> org.apache.spark.sql.hive.orc.OrcFileFormat, org.apache.spark.sql.execution.datasources.orc -> org.apache.spark.sql.execution.datasources.orc.OrcFileFormat, org.apache.spark.sql.jdbc -> org.apache.spark.sql.execution.datasources.jdbc.JdbcRelationProvider, org.apache.spark.sql.execution.datasources.jdbc.DefaultSource -> org.apache.spark.sql.execution.datasources.jdbc.JdbcRelationProvider, org.apache.spark.sql.execution.datasources.parquet -> org.apache.spark.sql.execution.datasources.parquet.ParquetFileFormat)
> ```

这个 Map 集合中并未找到 provider(即"jdbc"), 之后 Spark 使用了 Java SPI 机制, 加载了 `Trait DataSourceRegister` 的所有实现类。 `val serviceLoader = ServiceLoader.load(classOf[DataSourceRegister], loader)`

然后使用 Scala 的模式匹配找到 DataSourceRegister 的实现类的 shortName == "jdbc"。`serviceLoader.asScala.filter(_.shortName().equalsIgnoreCase(provider1)`。

这里最终过滤得到的是:`org.apache.spark.sql.execution.datasources.jdbc.JdbcRelationProvider`

之后的代码就进入到了 `DataFrameReader#loadV1Source()`

## 2. DataFrameReader#loadV1Source()

```scala
private def loadV1Source(paths: String*) = {
  // Code path for data source v1.
  sparkSession.baseRelationToDataFrame(
    DataSource.apply(
      sparkSession,
      paths = paths,
      userSpecifiedSchema = userSpecifiedSchema,
      className = source,
      options = extraOptions.toMap).resolveRelation())
}
```

这个方法代码较为简单: 内部调用一目了然 SparkSession#baseRelationToDataFrame() 就是将 BaseRelation 转换为 DataFrame(这里的 BaseRelation 就是 JdbcRelationProvider)。

在 SparkSession#baseRelationToDataFrame() 方法内部主要就是实例化了 DataSource, 之后调用 resolveRelation() 方法。

OK, 我们仅需跟进 DataSource 类的构造方法:



## 3. DataSource

```scala
case class DataSource(
  // SparkSession
  sparkSession: SparkSession,
  // jdbc 
  className: String,
  // null
  paths: Seq[String] = Nil,
  // null
  userSpecifiedSchema: Option[StructType] = None,
  partitionColumns: Seq[String] = Seq.empty,
  bucketSpec: Option[BucketSpec] = None,
  options: Map[String, String] = Map.empty,
  catalogTable: Option[CatalogTable] = None) extends Logging {

  // ...... 省略代码无数

  // 就看这个最重要的方法 resolveRelation 
  def resolveRelation(checkFilesExist: Boolean = true): BaseRelation = {	
		// providingClass 这个变量也是通过 lookupDataSource() 方法最中取得 JdbcRelationProvider
    val relation = (providingClass.newInstance(), userSpecifiedSchema) match {
      // TODO: Throw when too much is given.
      case (dataSource: SchemaRelationProvider, Some(schema)) =>
      dataSource.createRelation(sparkSession.sqlContext, caseInsensitiveOptions, schema)
      case (dataSource: RelationProvider, None) =>
      dataSource.createRelation(sparkSession.sqlContext, caseInsensitiveOptions)
      case (_: SchemaRelationProvider, None) =>
      throw new AnalysisException(s"A schema needs to be specified when using $className.")
      case (dataSource: RelationProvider, Some(schema)) =>
      val baseRelation =
      dataSource.createRelation(sparkSession.sqlContext, caseInsensitiveOptions)
      if (baseRelation.schema != schema) {
        throw new AnalysisException(s"$className does not allow user-specified schemas.")
      }
      baseRelation

      // We are reading from the results of a streaming query. Load files from the metadata log
      // instead of listing them using HDFS APIs.
      case (format: FileFormat, _)
      if FileStreamSink.hasMetadata(
        caseInsensitiveOptions.get("path").toSeq ++ paths,
        sparkSession.sessionState.newHadoopConf()) =>
      val basePath = new Path((caseInsensitiveOptions.get("path").toSeq ++ paths).head)
      val tempFileCatalog = new MetadataLogFileIndex(sparkSession, basePath, None)
      val fileCatalog = if (userSpecifiedSchema.nonEmpty) {
        val partitionSchema = combineInferredAndUserSpecifiedPartitionSchema(tempFileCatalog)
        new MetadataLogFileIndex(sparkSession, basePath, Option(partitionSchema))
      } else {
        tempFileCatalog
      }
      val dataSchema = userSpecifiedSchema.orElse {
        format.inferSchema(
          sparkSession,
          caseInsensitiveOptions,
          fileCatalog.allFiles())
      }.getOrElse {
        throw new AnalysisException(
          s"Unable to infer schema for $format at ${fileCatalog.allFiles().mkString(",")}. " +
          "It must be specified manually")
      }

      HadoopFsRelation(
        fileCatalog,
        partitionSchema = fileCatalog.partitionSchema,
        dataSchema = dataSchema,
        bucketSpec = None,
        format,
        caseInsensitiveOptions)(sparkSession)

      // This is a non-streaming file based datasource.
      case (format: FileFormat, _) =>
      val allPaths = caseInsensitiveOptions.get("path") ++ paths
      val hadoopConf = sparkSession.sessionState.newHadoopConf()
      val globbedPaths = allPaths.flatMap(
        DataSource.checkAndGlobPathIfNecessary(hadoopConf, _, checkFilesExist)).toArray

      val fileStatusCache = FileStatusCache.getOrCreate(sparkSession)
      val (dataSchema, partitionSchema) = getOrInferFileFormatSchema(format, fileStatusCache)

      val fileCatalog = if (sparkSession.sqlContext.conf.manageFilesourcePartitions &&
                            catalogTable.isDefined && catalogTable.get.tracksPartitionsInCatalog) {
        val defaultTableSize = sparkSession.sessionState.conf.defaultSizeInBytes
        new CatalogFileIndex(
          sparkSession,
          catalogTable.get,
          catalogTable.get.stats.map(_.sizeInBytes.toLong).getOrElse(defaultTableSize))
      } else {
        new InMemoryFileIndex(
          sparkSession, globbedPaths, options, Some(partitionSchema), fileStatusCache)
      }

      HadoopFsRelation(
        fileCatalog,
        partitionSchema = partitionSchema,
        dataSchema = dataSchema.asNullable,
        bucketSpec = bucketSpec,
        format,
        caseInsensitiveOptions)(sparkSession)

      case _ =>
      throw new AnalysisException(
        s"$className is not a valid Spark SQL Data Source.")
    }

    relation match {
      case hs: HadoopFsRelation =>
      SchemaUtils.checkColumnNameDuplication(
        hs.dataSchema.map(_.name),
        "in the data schema",
        equality)
      SchemaUtils.checkColumnNameDuplication(
        hs.partitionSchema.map(_.name),
        "in the partition schema",
        equality)
      case _ =>
      SchemaUtils.checkColumnNameDuplication(
        relation.schema.map(_.name),
        "in the data schema",
        equality)
    }

    relation
  }
}
```

在 resolveRelation() 方法中有一个关键变量 providingClass, 它是成员变量, 它的值也是通过 方法 lookupDataSource() 方法最中取得的, 所以这个变量值在当前语境中就是 JdbcRelationProvider。

之后就进入了 case 判断, 最终进入的判断逻辑就是

```scala
case (dataSource: RelationProvider, None) =>
  dataSource.createRelation(sparkSession.sqlContext, caseInsensitiveOptions)
```

这里的的 dataSource 的实现类就是 JdbcRelationProvider, 所以代码真正进入的是 `JdbcRelationProvider#createRelation()`

## 4. JdbcRelationProvider#createRelation()

```scala
override def createRelation(
  sqlContext: SQLContext,
  parameters: Map[String, String]): BaseRelation = {
  import JDBCOptions._

  val jdbcOptions = new JDBCOptions(parameters)
  val partitionColumn = jdbcOptions.partitionColumn
  val lowerBound = jdbcOptions.lowerBound
  val upperBound = jdbcOptions.upperBound
  val numPartitions = jdbcOptions.numPartitions

  val partitionInfo = if (partitionColumn.isEmpty) {
    assert(lowerBound.isEmpty && upperBound.isEmpty, "When 'partitionColumn' is not specified, " +
           s"'$JDBC_LOWER_BOUND' and '$JDBC_UPPER_BOUND' are expected to be empty")
    null
  } else {
    assert(lowerBound.nonEmpty && upperBound.nonEmpty && numPartitions.nonEmpty,
           s"When 'partitionColumn' is specified, '$JDBC_LOWER_BOUND', '$JDBC_UPPER_BOUND', and " +
           s"'$JDBC_NUM_PARTITIONS' are also required")
    
    JDBCPartitioningInfo(partitionColumn.get, 
                         lowerBound.get, 
                         upperBound.get, 
                         numPartitions.get)
  }
  val parts = JDBCRelation.columnPartition(partitionInfo)
  JDBCRelation(parts, jdbcOptions)(sqlContext.sparkSession)
}
```

代码中主要都是进行一些变量初始化, 真正的执行逻辑在最后两行块:

- `JDBCPartitioningInfo(partitionColumn.get,` 
                           `lowerBound.get,` 
                           `upperBound.get,` 
                           `numPartitions.get)`

-  `JDBCRelation(parts, jdbcOptions)(sqlContext.sparkSession)`

这里暂时只分析第二个方法。

### 4.1 JDBCRelation

```scala
private[sql] case class JDBCRelation(
  parts: Array[Partition], jdbcOptions: JDBCOptions)(@transient val sparkSession: SparkSession)
extends BaseRelation
with PrunedFilteredScan
with InsertableRelation {

  override def sqlContext: SQLContext = sparkSession.sqlContext

  override val needConversion: Boolean = false

  override val schema: StructType = {
    val tableSchema = JDBCRDD.resolveTable(jdbcOptions)
    jdbcOptions.customSchema match {
      case Some(customSchema) => JdbcUtils.getCustomSchema(
        tableSchema, customSchema, sparkSession.sessionState.conf.resolver)
      case None => tableSchema
    }
  }

  // Check if JDBCRDD.compileFilter can accept input filters
  override def unhandledFilters(filters: Array[Filter]): Array[Filter] = {
    filters.filter(JDBCRDD.compileFilter(_, JdbcDialects.get(jdbcOptions.url)).isEmpty)
  }

  override def buildScan(requiredColumns: Array[String], filters: Array[Filter]): RDD[Row] = {
    // Rely on a type erasure hack to pass RDD[InternalRow] back as RDD[Row]
    JDBCRDD.scanTable(
      sparkSession.sparkContext,
      schema,
      requiredColumns,
      filters,
      parts,
      jdbcOptions).asInstanceOf[RDD[Row]]
  }

  override def insert(data: DataFrame, overwrite: Boolean): Unit = {
    data.write
    .mode(if (overwrite) SaveMode.Overwrite else SaveMode.Append)
    .jdbc(jdbcOptions.url, jdbcOptions.table, jdbcOptions.asProperties)
  }

  override def toString: String = {
    val partitioningInfo = if (parts.nonEmpty) s" [numPartitions=${parts.length}]" else ""
    // credentials should not be included in the plan output, table information is sufficient.
    s"JDBCRelation(${jdbcOptions.table})" + partitioningInfo
  }
}
```

这里就一切明朗了, `JDBCRelation(...) extends BaseRelation`, 所以这里就返回了一个 BaseRelation 的实现类作为参数给方法 `DataFrameReader#loadV1Source()` 中的 `sparkSession#baseRelationToDataFrame(BaseRelation)`, 之后就可以将 BaseRelation 转换为 DataFrame。

这里简单总结下: baseRelation -> logicalPlan -> executePlan -> Dataset[Row] (DataFrame)

从baseRelation到logicalPlan是由

org.apache.spark.sql.execution.datasources包下的

LogicalRelation类处理得到的。

注释翻译：

LogicalRelation 类用来把 BaseRelation 链接到(link into) 一个逻辑 查询 计划(logical query plan)

## 5. 如何获取 JDBC Schema

到上面第四步基本逻辑就都走通了, 那么回到最初的问题, Spark 是如何获取 JDBC 表的 Schema 的呢?

这个部分逻辑就在 `JDBCRelation#schema()` 方法中 `val tableSchema = JDBCRDD.resolveTable(jdbcOptions)`

```scala
def resolveTable(options: JDBCOptions): StructType = {
  val url = options.url
  val table = options.table
  val dialect = JdbcDialects.get(url)
  val conn: Connection = JdbcUtils.createConnectionFactory(options)()
  try {
    // dialect.getSchemaQuery(table), 这个其实就是 "SELECT * FROM ${table} WHERE 1=0"
    val statement = conn.prepareStatement(dialect.getSchemaQuery(table))
    try {
      val rs = statement.executeQuery()
      try {
        JdbcUtils.getSchema(rs, dialect, alwaysNullable = true)
      } finally {
        rs.close()
      }
    } finally {
      statement.close()
    }
  } finally {
    conn.close()
  }
}
```

这里就很清楚的可以看到, 其实获取 Schema 的逻辑就是通过 JDBC 执行一条 SQL 查询语句 `"SELECT * FROM $table WHERE 1=0"`, 获取对应 SQL 的 PrepareStatement 然后执行 executeQuery 来获取 ResultSet,之后通过`JdbcUtils.getSchema(rs, dialect, alwaysNullable = true)` 来获取 Spark Schema(正确的说应该是 StructType)

### 5.1 JdbcUtils#getSchema

```scala
def getSchema(
  resultSet: ResultSet,
  dialect: JdbcDialect,
  alwaysNullable: Boolean = false): StructType = {
  val rsmd = resultSet.getMetaData
  val ncols = rsmd.getColumnCount
  val fields = new Array[StructField](ncols)
  var i = 0
  while (i < ncols) {
    val columnName = rsmd.getColumnLabel(i + 1)
    val dataType = rsmd.getColumnType(i + 1)
    val typeName = rsmd.getColumnTypeName(i + 1)
    val fieldSize = rsmd.getPrecision(i + 1)
    val fieldScale = rsmd.getScale(i + 1)
    val isSigned = {
      try {
        rsmd.isSigned(i + 1)
      } catch {
        // Workaround for HIVE-14684:
        case e: SQLException if
        e.getMessage == "Method not supported" &&
        rsmd.getClass.getName == "org.apache.hive.jdbc.HiveResultSetMetaData" => true
      }
    }
    val nullable = if (alwaysNullable) {
      true
    } else {
      rsmd.isNullable(i + 1) != ResultSetMetaData.columnNoNulls
    }
    val metadata = new MetadataBuilder().putLong("scale", fieldScale)
    val columnType =
    dialect.getCatalystType(dataType, typeName, fieldSize, metadata).getOrElse(
      getCatalystType(dataType, fieldSize, fieldScale, isSigned))
    fields(i) = StructField(columnName, columnType, nullable)
    i = i + 1
  }
  new StructType(fields)
}
```

