## 【FLINK源码】flink1.10 解析SQL

```sql
-- -- 开启 mini-batch
-- SET table.exec.mini-batch.enabled=true;
-- -- mini-batch的时间间隔，即作业需要额外忍受的延迟
-- SET table.exec.mini-batch.allow-latency=1s;
-- -- 一个 mini-batch 中允许最多缓存的数据
-- SET table.exec.mini-batch.size=1000;
-- -- 开启 local-global 优化
-- SET table.optimizer.agg-phase-strategy=TWO_PHASE;
--
-- -- 开启 distinct agg 切分
-- SET table.optimizer.distinct-agg.split.enabled=true;


-- source
CREATE TABLE user_log (
    user_id VARCHAR,
    item_id VARCHAR,
    category_id VARCHAR,
    behavior VARCHAR,
    ts TIMESTAMP
) WITH (
    'connector.type' = 'kafka',
    'connector.version' = 'universal',
    'connector.topic' = 'user_behavior',
    'connector.startup-mode' = 'earliest-offset',
    'connector.properties.0.key' = 'zookeeper.connect',
    'connector.properties.0.value' = 'localhost:2181',
    'connector.properties.1.key' = 'bootstrap.servers',
    'connector.properties.1.value' = 'localhost:9092',
    'update-mode' = 'append',
    'format.type' = 'json',
    'format.derive-schema' = 'true'
);

-- sink
CREATE TABLE pvuv_sink (
    dt VARCHAR,
    pv BIGINT,
    uv BIGINT
) WITH (
    'connector.type' = 'jdbc',
    'connector.url' = 'jdbc:mysql://localhost:3306/flink-test',
    'connector.table' = 'pvuv_sink',
    'connector.username' = 'root',
    'connector.password' = '123456',
    'connector.write.flush.max-rows' = '1'
);


INSERT INTO pvuv_sink
SELECT
  DATE_FORMAT(ts, 'yyyy-MM-dd HH:00') dt,
  COUNT(*) AS pv,
  COUNT(DISTINCT user_id) AS uv
FROM user_log
GROUP BY DATE_FORMAT(ts, 'yyyy-MM-dd HH:00');
```

Flink从1.9开始支持DDL, 解析DDL-SQL的代码入口方法: **tEnv.sqlUpdate(ddl);**

```java
private void callCreateTable(SqlCommandCall cmdCall) {
  String ddl = cmdCall.operands[0];
  try {
    tEnv.sqlUpdate(ddl);
  } catch (SqlParserException e) {
    throw new RuntimeException("SQL parse failed:\n" + ddl + "\n", e);
  }
}
```

**tEnv.sqlUpdate(ddl)** 这个方法位于***org.apache.flink.table.api.internal.TableEnvironmentImpl # sqlUpdate***

```java
public void sqlUpdate(String stmt) {
		List<Operation> operations = parser.parse(stmt);

		if (operations.size() != 1) {
			throw new TableException(UNSUPPORTED_QUERY_IN_SQL_UPDATE_MSG);
		}

		Operation operation = operations.get(0);

		if (operation instanceof ModifyOperation) {
			List<ModifyOperation> modifyOperations = Collections.singletonList((ModifyOperation) operation);
			if (isEagerOperationTranslation()) {
				translate(modifyOperations);
			} else {
				buffer(modifyOperations);
			}
		} else if (operation instanceof CreateTableOperation) {
			CreateTableOperation createTableOperation = (CreateTableOperation) operation;
			catalogManager.createTable(
        createTableOperation.getCatalogTable(),
        createTableOperation.getTableIdentifier(),
        createTableOperation.isIgnoreIfExists());
		} else if (operation instanceof CreateDatabaseOperation) {
			CreateDatabaseOperation createDatabaseOperation = (CreateDatabaseOperation) operation;
			Catalog catalog = getCatalogOrThrowException(createDatabaseOperation.getCatalogName());
			String exMsg = getDDLOpExecuteErrorMsg(createDatabaseOperation.asSummaryString());
			try {
				catalog.createDatabase(
          createDatabaseOperation.getDatabaseName(),
          createDatabaseOperation.getCatalogDatabase(),
          createDatabaseOperation.isIgnoreIfExists());
			} catch (DatabaseAlreadyExistException e) {
				throw new ValidationException(exMsg, e);
			} catch (Exception e) {
				throw new TableException(exMsg, e);
			}
		} else if (operation instanceof DropTableOperation) {
			DropTableOperation dropTableOperation = (DropTableOperation) operation;
			catalogManager.dropTable(
        dropTableOperation.getTableIdentifier(),
        dropTableOperation.isIfExists());
		} else if (operation instanceof AlterTableOperation) {
			AlterTableOperation alterTableOperation = (AlterTableOperation) operation;
			Catalog catalog = getCatalogOrThrowException(alterTableOperation.getTableIdentifier().getCatalogName());
			String exMsg = getDDLOpExecuteErrorMsg(alterTableOperation.asSummaryString());
			try {
				if (alterTableOperation instanceof AlterTableRenameOperation) {
					AlterTableRenameOperation alterTableRenameOp = (AlterTableRenameOperation) operation;
					catalog.renameTable(
            alterTableRenameOp.getTableIdentifier().toObjectPath(),
            alterTableRenameOp.getNewTableIdentifier().getObjectName(),
            false);
				} else if (alterTableOperation instanceof AlterTablePropertiesOperation){
					AlterTablePropertiesOperation alterTablePropertiesOp = (AlterTablePropertiesOperation) operation;
					catalog.alterTable(
            alterTablePropertiesOp.getTableIdentifier().toObjectPath(),
            alterTablePropertiesOp.getCatalogTable(),
            false);
				}
			} catch (TableAlreadyExistException | TableNotExistException e) {
				throw new ValidationException(exMsg, e);
			} catch (Exception e) {
				throw new TableException(exMsg, e);
			}
		} else if (operation instanceof DropDatabaseOperation) {
			DropDatabaseOperation dropDatabaseOperation = (DropDatabaseOperation) operation;
			Catalog catalog = getCatalogOrThrowException(dropDatabaseOperation.getCatalogName());
			String exMsg = getDDLOpExecuteErrorMsg(dropDatabaseOperation.asSummaryString());
			try {
				catalog.dropDatabase(
          dropDatabaseOperation.getDatabaseName(),
          dropDatabaseOperation.isIfExists(),
          dropDatabaseOperation.isCascade());
			} catch (DatabaseNotExistException | DatabaseNotEmptyException e) {
				throw new ValidationException(exMsg, e);
			} catch (Exception e) {
				throw new TableException(exMsg, e);
			}
		} else if (operation instanceof AlterDatabaseOperation) {
			AlterDatabaseOperation alterDatabaseOperation = (AlterDatabaseOperation) operation;
			Catalog catalog = getCatalogOrThrowException(alterDatabaseOperation.getCatalogName());
			String exMsg = getDDLOpExecuteErrorMsg(alterDatabaseOperation.asSummaryString());
			try {
				catalog.alterDatabase(
          alterDatabaseOperation.getDatabaseName(),
          alterDatabaseOperation.getCatalogDatabase(),
          false);
			} catch (DatabaseNotExistException e) {
				throw new ValidationException(exMsg, e);
			} catch (Exception e) {
				throw new TableException(exMsg, e);
			}
		} else if (operation instanceof CreateFunctionOperation) {
			CreateFunctionOperation createFunctionOperation = (CreateFunctionOperation) operation;
			createCatalogFunction(createFunctionOperation);
		} else if (operation instanceof CreateTempSystemFunctionOperation) {
			CreateTempSystemFunctionOperation createtempSystemFunctionOperation =
				(CreateTempSystemFunctionOperation) operation;
			createSystemFunction(createtempSystemFunctionOperation);
		} else if (operation instanceof AlterFunctionOperation) {
			AlterFunctionOperation alterFunctionOperation = (AlterFunctionOperation) operation;
			alterCatalogFunction(alterFunctionOperation);
		} else if (operation instanceof DropFunctionOperation) {
			DropFunctionOperation dropFunctionOperation = (DropFunctionOperation) operation;
			dropCatalogFunction(dropFunctionOperation);
		} else if (operation instanceof DropTempSystemFunctionOperation) {
			DropTempSystemFunctionOperation dropTempSystemFunctionOperation =
				(DropTempSystemFunctionOperation) operation;
			dropSystemFunction(dropTempSystemFunctionOperation);
		} else if (operation instanceof UseCatalogOperation) {
			UseCatalogOperation useCatalogOperation = (UseCatalogOperation) operation;
			catalogManager.setCurrentCatalog(useCatalogOperation.getCatalogName());
		} else if (operation instanceof UseDatabaseOperation) {
			UseDatabaseOperation useDatabaseOperation = (UseDatabaseOperation) operation;
			catalogManager.setCurrentCatalog(useDatabaseOperation.getCatalogName());
			catalogManager.setCurrentDatabase(useDatabaseOperation.getDatabaseName());
		} else {
			throw new TableException(UNSUPPORTED_QUERY_IN_SQL_UPDATE_MSG);
		}
	}
```

### TableEnvironmentImpl

```java

```



### Parser

***org.apache.flink.table.planner.delegation.PaeserImpl # parser()***

ParserImpl主要使用了calcite来解析SQL

ParserImpl的构造函数:

```java
public ParserImpl(
  CatalogManager catalogManager,
  Supplier<FlinkPlannerImpl> validatorSupplier,
  Supplier<CalciteParser> calciteParserSupplier) {
  this.catalogManager = catalogManager;
  this.validatorSupplier = validatorSupplier;
  this.calciteParserSupplier = calciteParserSupplier;
}
```

因为