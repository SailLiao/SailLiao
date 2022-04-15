---
title: Spring 异常捕获后依旧回滚
date: 2022-04-07 09:17:22
tags: Spring
cover:  https://w.wallhaven.cc/full/6o/wallhaven-6ozkzl.jpg
---

# 问题抛出

在 Spring 的声明式事务中手动捕获异常，居然判定回滚了，例如 A 里面调用 B ，C ，C 里面抛出了异常，A 里面对 C 进行了 try catch 依然会回滚，上代码

```JAVA
@EnableTransactionManagement
@SpringBootApplication
public class BingApplication {
    public static void main(String[] args) {
        SpringApplication.run(BingApplication.class, args);
    }
}

@Controller
public class IndexController {
    @Autowired
    ServiceA serviceA;

    @RequestMapping("test")
    @ResponseBody
    public String test() {
        serviceA.operate();
        return "OK";
    }

}


@Service
public class ServiceA {

    @Autowired
    private ServiceB serviceB;

    @Autowired
    private ServiceC serviceC;

    @Transactional
    public void operate() {
        serviceB.insertB();
        try {
            serviceC.insertC();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

@Service
public class ServiceB {

    @Autowired
    ImageMapper imageMapper;

    @Transactional
    public void insertB() {
        Image image = new Image();
        image.setDescription("descriptionb");
        image.setDate("dateb");
        image.setUrl("urlb");
        imageMapper.insert(image);
    }
}

@Service
public class ServiceC {

    @Autowired
    ImageMapper imageMapper;

    @Transactional
    public void insertC() {
        Image image = new Image();
        image.setDescription("descriptionc");
        image.setDate("datec");
        image.setUrl("urlc");
        imageMapper.insert(image);
        throw new RuntimeException();
    }
}

```

正常情况下，因为我们对 insertC 进行了 try catch，所以我们期望 b 都能插入数据成功，但是都没插入成功，证明被回滚了
@Transactional 默认的传播机制是 Propagation.REQUIRED，REQUIRED 的意思是当前没有事物就创建新的事物，当前有事物
就加入事物。

# 源码分析

事物的代理入口类是 TransactionInterceptor，该类通过 ProxyTransactionManagementConfiguration 注册到容器中
```JAVA
@Bean
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public TransactionInterceptor transactionInterceptor(TransactionAttributeSource transactionAttributeSource) {
	TransactionInterceptor interceptor = new TransactionInterceptor();
	interceptor.setTransactionAttributeSource(transactionAttributeSource);
	if (this.txManager != null) {
		interceptor.setTransactionManager(this.txManager);
	}
	return interceptor;
}
```

里面重要的是 invoke 方法，invoke 方法中重要的是 invokeWithinTransaction 方法

```JAVA
@Nullable
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
		final InvocationCallback invocation) throws Throwable {

	// If the transaction attribute is null, the method is non-transactional.
	TransactionAttributeSource tas = getTransactionAttributeSource();
	final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
	final TransactionManager tm = determineTransactionManager(txAttr);

	。。。。。。

	PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
	final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

	// 注解的事务会走这里
	if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
		// 开启事务
		// Standard transaction demarcation with getTransaction and commit/rollback calls.
		TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

		Object retVal;
		try {
			// This is an around advice: Invoke the next interceptor in the chain.
			// This will normally result in a target object being invoked.
			retVal = invocation.proceedWithInvocation();
		}
		catch (Throwable ex) {
			// target invocation exception
			// 回滚
			completeTransactionAfterThrowing(txInfo, ex);
			throw ex;
		}
		finally {
			cleanupTransactionInfo(txInfo);
		}

		if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
			// Set rollback-only in case of Vavr failure matching our rollback rules...
			TransactionStatus status = txInfo.getTransactionStatus();
			if (status != null && txAttr != null) {
				retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
			}
		}

        // 提交事务
		commitTransactionAfterReturning(txInfo);
		return retVal;
	}

	。。。。。。
}
```

createTransactionIfNecessary 中有个 getTransaction 

```JAVA
@Override
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
		throws TransactionException {

	// Use defaults if no transaction definition given.
	TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());

	Object transaction = doGetTransaction();
	boolean debugEnabled = logger.isDebugEnabled();

	// 如果存在事务
	if (isExistingTransaction(transaction)) {
		// 处理存在事务的情况
		// Existing transaction found -> check propagation behavior to find out how to behave.
		return handleExistingTransaction(def, transaction, debugEnabled);
	}

	// Check definition settings for new transaction.
	if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
		throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
	}

	// No existing transaction found -> check propagation behavior to find out how to proceed.
	if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
		throw new IllegalTransactionStateException(
				"No existing transaction found for transaction marked with propagation 'mandatory'");
	}
	// 第一次大部分走这里
	else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
			def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
			def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
		SuspendedResourcesHolder suspendedResources = suspend(null);
		if (debugEnabled) {
			logger.debug("Creating new transaction with name [" + def.getName() + "]: " + def);
		}
		try {
			return startTransaction(def, transaction, debugEnabled, suspendedResources);
		}
		catch (RuntimeException | Error ex) {
			resume(null, suspendedResources);
			throw ex;
		}
	}
	else {
		// Create "empty" transaction: no actual transaction, but potentially synchronization.
		if (def.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
			logger.warn("Custom isolation level specified but no actual transaction initiated; " +
					"isolation level will effectively be ignored: " + def);
		}
		boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
		return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
	}
}
```

我们第一次进来的时候是没有事务的，所以会新建事务 startTransaction

```JAVA
private TransactionStatus startTransaction(TransactionDefinition definition, Object transaction,
		boolean debugEnabled, @Nullable SuspendedResourcesHolder suspendedResources) {

	boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
	// 开启一个新的事务，这里有的参数 newTransaction  传入的是 true
	DefaultTransactionStatus status = newTransactionStatus(
			definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
	doBegin(transaction, definition);
	prepareSynchronization(status, definition);
	return status;
}
```

第二次进来，因为有事务了，就会走到 handleExistingTransaction, 默认的 REQUIRED 就末尾的这一行

```JAVA
return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
```

这里是传入的是 false，最后，我们返回 commitTransactionAfterReturning 方法中，看 commit 方法 中的 processCommit

```JAVA
// 如果有回滚点
if (status.hasSavepoint()) {
	if (status.isDebug()) {
		logger.debug("Releasing transaction savepoint");
	}
	unexpectedRollback = status.isGlobalRollbackOnly();
	status.releaseHeldSavepoint();
}
// 如果是新的事务
else if (status.isNewTransaction()) {
	if (status.isDebug()) {
		logger.debug("Initiating transaction commit");
	}
	unexpectedRollback = status.isGlobalRollbackOnly();
	doCommit(status);
}
else if (isFailEarlyOnGlobalRollbackOnly()) {
	unexpectedRollback = status.isGlobalRollbackOnly();
}
```

因为 newTransaction 不为 true，我们看异常情况下的回滚 在 org.springframework.transaction.support.AbstractPlatformTransactionManager#commit() 方法中的 processRollback
最终会走到 **doSetRollbackOnly**

doSetRollbackOnly -> DataSourceTransactionObject.setRollbackOnly() -> ResourceHolderSupport.setRollbackOnly()

```JAVA
public void setRollbackOnly() {
	this.rollbackOnly = true;
}
```

所以最后我们提交的时候，在 AbstractPlatformTransactionManager#commit(TransactionStatus status) 方法中

```JAVA
if (defStatus.isLocalRollbackOnly()) {
	if (defStatus.isDebug()) {
		logger.debug("Transactional code has requested rollback");
	}
	processRollback(defStatus, false);
	return;
}
```

如果有 rollbackOnly = true ，所以都会回滚

# 总结和其他

## 不同的情况

如果 A 有事物，B 有事物，C 事务，在 A 里面调用 B、C，C抛出运行时异常，即使被 try catch，数据不会插入

如果 A 有事物，B 有事物，C 事务，在 A 里面调用 B、C，C抛出编译时异常，即使被 try catch，数据也会插入

### 运行时异常 编译时异常

比如 RuntimeException 时 运行时异常，IOException 是编译时异常

如果 A 有事物，B 有事物，C 无事务，在 A 里面调用 B、C，C抛出运行时异常，如果不对 C 进行 try catch，数据不会插入

如果 A 有事物，B 有事物，C 无事务，在 A 里面调用 B、C，C抛出运行时异常，如果对 C 进行 try catch，数据都会插入

# 来源

https://blog.csdn.net/weixin_64314555/article/details/122492760