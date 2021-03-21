## Flink如何从jar中获取StreamGraph

### overview

整段代码逻辑大致:  invoke() --> 用户main --> throw ProgramAbortException --> StreamGraph --> JobGraph --> 提交



### StreamGraph

执行用户jar逻辑是发生在从用户代码中获取***StreamGraph*** ( ***FlinkPlan***接口的实现类 )的过程中

```java
public static JobGraph createJobGraph(
			PackagedProgram packagedProgram,
			Configuration configuration,
			int defaultParallelism,
			@Nullable JobID jobID) throws ProgramInvocationException {
		Thread.currentThread().setContextClassLoader(packagedProgram.getUserCodeClassLoader());
		final Optimizer optimizer = new Optimizer(new DataStatistics(), new DefaultCostEstimator(), configuration);
		final FlinkPlan flinkPlan;

		if (packagedProgram.isUsingProgramEntryPoint()) {

			final JobWithJars jobWithJars = packagedProgram.getPlanWithJars();

			final Plan plan = jobWithJars.getPlan();

			if (plan.getDefaultParallelism() <= 0) {
				plan.setDefaultParallelism(defaultParallelism);
			}

			flinkPlan = optimizer.compile(jobWithJars.getPlan());
      // 使用命令行提交作业代码会进入这个if分支
		} else if (packagedProgram.isUsingInteractiveMode()) {
			final OptimizerPlanEnvironment optimizerPlanEnvironment = new OptimizerPlanEnvironment(optimizer);

			optimizerPlanEnvironment.setParallelism(defaultParallelism);

			flinkPlan = optimizerPlanEnvironment.getOptimizedPlan(packagedProgram);
		} else {
			throw new ProgramInvocationException("PackagedProgram does not have a valid invocation mode.");
		}

		final JobGraph jobGraph;

		if (flinkPlan instanceof StreamingPlan) {
			jobGraph = ((StreamingPlan) flinkPlan).getJobGraph(jobID);
			jobGraph.setSavepointRestoreSettings(packagedProgram.getSavepointSettings());
		} else {
			final JobGraphGenerator jobGraphGenerator = new JobGraphGenerator(configuration);
			jobGraph = jobGraphGenerator.compileJobGraph((OptimizedPlan) flinkPlan, jobID);
		}

		for (URL url : packagedProgram.getAllLibraries()) {
			try {
				jobGraph.addJar(new Path(url.toURI()));
			} catch (URISyntaxException e) {
				throw new ProgramInvocationException("Invalid URL for jar file: " + url + '.', jobGraph.getJobID(), e);
			}
		}

		jobGraph.setClasspaths(packagedProgram.getClasspaths());

		return jobGraph;
	}
```

当使用命令行模式提交作业代码会进入如下分支

```java
else if (packagedProgram.isUsingInteractiveMode()) {
			final OptimizerPlanEnvironment optimizerPlanEnvironment = new OptimizerPlanEnvironment(optimizer);

			optimizerPlanEnvironment.setParallelism(defaultParallelism);

			flinkPlan = optimizerPlanEnvironment.getOptimizedPlan(packagedProgram);
		}
```

1. 利用***optimizer***构造***OptimizerPlanEnvironment*** 实例 ***optimizerPlanEnvironment***

2. 获取***StreamGraph***的逻辑:

   ***flinkPlan = optimizerPlanEnvironment.getOptimizedPlan(packagedProgram);***

   ```java
   public FlinkPlan getOptimizedPlan(PackagedProgram prog) throws ProgramInvocationException {
   
   		// temporarily write syserr and sysout to a byte array.
   		PrintStream originalOut = System.out;
   		PrintStream originalErr = System.err;
   		ByteArrayOutputStream baos = new ByteArrayOutputStream();
   		System.setOut(new PrintStream(baos));
   		ByteArrayOutputStream baes = new ByteArrayOutputStream();
   		System.setErr(new PrintStream(baes));
   
   		setAsContext();
   		try {
   			prog.invokeInteractiveModeForExecution();
   		}
   		catch (ProgramInvocationException e) {
   			throw e;
   		}
   		catch (Throwable t) {
   			// the invocation gets aborted with the preview plan
   			if (optimizerPlan != null) {
   				return optimizerPlan;
   			} else {
   				throw new ProgramInvocationException("The program caused an error: ", t);
   			}
   		}
   		finally {
   			unsetAsContext();
   			System.setOut(originalOut);
   			System.setErr(originalErr);
   		}
   
   		String stdout = baos.toString();
   		String stderr = baes.toString();
   
   		throw new ProgramInvocationException(
   				"The program plan could not be fetched - the program aborted pre-maturely."
   						+ "\n\nSystem.err: " + (stdout.length() == 0 ? "(none)" : stdout)
   						+ "\n\nSystem.out: " + (stderr.length() == 0 ? "(none)" : stderr));
   	}
   ```

3. 关注一个方法***setAsContext();***

   ```java
   private void setAsContext() {
   		ExecutionEnvironmentFactory factory = new ExecutionEnvironmentFactory() {
   
   			@Override
   			public ExecutionEnvironment createExecutionEnvironment() {
   				return OptimizerPlanEnvironment.this;
   			}
   		};
   		initializeContextEnvironment(factory);
   	}
   ```

   这个方法会将步骤2中的***OptimizerPlanEnvironment*** 的静态变量 ***ExecutionEnvironmentFactory contextEnvironmentFactory***赋值,  所以之后 ***contextEnvironmentFactory*** 这个已经有值了

4. 进入步骤2代码中的***prog.invokeInteractiveModeForExecution();***

   ```Java
   public void invokeInteractiveModeForExecution() throws ProgramInvocationException{
   		if (isUsingInteractiveMode()) {
   			callMainMethod(mainClass, args);
   		} else {
   			throw new ProgramInvocationException("Cannot invoke a plan-based program directly.");
   		}
   	}
   ```

   继续跟进 ***callMainMethod(mainClass, args);***

   ```java
   private static void callMainMethod(Class<?> entryClass, String[] args) throws ProgramInvocationException {
   		Method mainMethod;
   		if (!Modifier.isPublic(entryClass.getModifiers())) {
   			throw new ProgramInvocationException("The class " + entryClass.getName() + " must be public.");
   		}
   
   		try {
   			mainMethod = entryClass.getMethod("main", String[].class);
   		} catch (NoSuchMethodException e) {
   			throw new ProgramInvocationException("The class " + entryClass.getName() + " has no main(String[]) method.");
   		}
   		catch (Throwable t) {
   			throw new ProgramInvocationException("Could not look up the main(String[]) method from the class " +
   					entryClass.getName() + ": " + t.getMessage(), t);
   		}
   		
     // 省略一堆try{} catch{}
     ......
   	
   }
   ```

   代码从这个方法就开始进入用户编写的 jar 中代码逻辑了, 所以我们进入Flink官网示例wordCount中继续跟.

### 用户jar

5. ***StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();***

   ```java
   public static StreamExecutionEnvironment getExecutionEnvironment() {
   		if (contextEnvironmentFactory != null) {
   			return contextEnvironmentFactory.createExecutionEnvironment();
   		}
   
   		// because the streaming project depends on "flink-clients" (and not the other way around)
   		// we currently need to intercept the data set environment and create a dependent stream env.
   		// this should be fixed once we rework the project dependencies
   
   		ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
   		if (env instanceof ContextEnvironment) {
   			return new StreamContextEnvironment((ContextEnvironment) env);
   		} else if (env instanceof OptimizerPlanEnvironment || env instanceof PreviewPlanEnvironment) {
   			return new StreamPlanEnvironment(env);
   		} else {
   			return createLocalEnvironment();
   		}
   	}
   ```

   这里拿到的 ***env*** 就是步骤3 中的***OptimizerPlanEnvironment.this***;

   如下 步骤3 中 ***setAsContext()*** 方法中将接口 ***ExecutionEnvironmentFactory*** 实例化并指向了抽象父类 ***ExecutionEnvironment.contextEnvironmentFactory*** 属性

   ```java
   public static ExecutionEnvironment getExecutionEnvironment() {
   	return contextEnvironmentFactory == null ?createLocalEnvironment() : contextEnvironmentFactory.createExecutionEnvironment();
   }
   ```

   所以上面的代码 **contextEnvironmentFactory == null **就为 false , 此时返回的 值就是 ***contextEnvironmentFactory.createExecutionEnvironment();***

   这方法就是 ***ExecutionEnvironmentFactory*** 接口中的 ***createExecutionEnvironment()*** 也就是步骤3 ***setAsContext()*** 中实现的方法 , 所以最后方法返回值就是 ***OptimizerPlanEnvironment.this***;

6. 所以最后命令行提交的情况下, 在用户代码中拿到的 ***ExecutionEnvironment*** 就是

   ***StreamPlanEnvironment( OptimizerPlanEnvironment.this )*** 的实例;

7. 接着我们来跟执行方法 ***execute()***

8. 第6步骤中已经说明 当前环境是 ***StreamPlanEnvironment*** , 所以跟进***StreamPlanEnvironment.execute() *** 

   ```java
   @Override
   public JobExecutionResult execute(String jobName) throws Exception {
     
   	StreamGraph streamGraph = getStreamGraph();
   	streamGraph.setJobName(jobName);
   
     transformations.clear();
   
     if (env instanceof OptimizerPlanEnvironment) {
       ((OptimizerPlanEnvironment) env).setPlan(streamGraph);
     } else if (env instanceof PreviewPlanEnvironment) {
       ((PreviewPlanEnvironment) env).setPreview(streamGraph.getStreamingPlanAsJSON());
     }
   
   	throw new OptimizerPlanEnvironment.ProgramAbortException();
   }
   ```

   ***((OptimizerPlanEnvironment) env).setPlan(streamGraph);*** 将用户的代码生成 ***StreamGraph*** 并且赋值到当前环境 ***(OptimizerPlanEnvironment)env.optimizerPlan*** 属性中

9. 用户逻辑代码执行完后会抛出 ***ProgramAbortException*** 异常, 然后会被 步骤2中捕获到, 最后获取 ***StreamGraph***

   ```java
   catch (Throwable t) {
     // the invocation gets aborted with the preview plan
     if (optimizerPlan != null) {
       return optimizerPlan;
     } else {
       throw new ProgramInvocationException("The program caused an error: ", t);
     }
   }
   ```

10. 

