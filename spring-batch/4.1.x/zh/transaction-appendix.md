## Appendix A：批处理和交易


### A.1 . 简单批处理，无需重试

请考虑以下没有重试的嵌套批处理的简单示例 . 它显示了批处理的常见场景：处理输入源直到耗尽，并且我们在处理的“块”结束时定期提交 . 


```java
1   |  REPEAT(until=exhausted) {
|
2   |    TX {
3   |      REPEAT(size=5) {
3.1 |        input;
3.2 |        output;
|      }
|    }
|
|  }
```


输入操作（3.1）可以是基于消息的接收（例如来自JMS），也可以是基于文件的读取，但要恢复并继续处理并有可能完成整个作业，它必须是事务性的 . 这同样适用于3.2的操作 . 它必须是事务性的或幂等的 . 

如果 `REPEAT` （3）处的块由于3.2处的数据库异常而失败，则 `TX` （2）必须回滚整个块 . 


### A.2 . 简单的无状态重试

对非事务性操作（例如对Web服务或其他远程资源的调用）使用重试也很有用，如以下示例所示：


```java
0   |  TX {
1   |    input;
1.1 |    output;
2   |    RETRY {
2.1 |      remote access;
|    }
|  }
```


这实际上是重试最有用的应用程序之一，因为远程调用比数据库更新更可能失败并且可以重试 . 只要远程访问（2.1）最终成功，事务 `TX` （0）就会提交 . 如果远程访问（2.1）最终失败，则保证事务 `TX` （0）回滚 . 


### A.3 . 典型的重复重试模式

最典型的批处理模式是向块的内部块添加重试，如以下示例所示：


```java
1   |  REPEAT(until=exhausted, exception=not critical) {
|
2   |    TX {
3   |      REPEAT(size=5) {
|
4   |        RETRY(stateful, exception=deadlock loser) {
4.1 |          input;
5   |        } PROCESS {
5.1 |          output;
6   |        } SKIP and RECOVER {
|          notify;
|        }
|
|      }
|    }
|
|  }
```


内部 `RETRY` （4）块标记为"stateful" . 有关有状态重试的说明，请参阅[the typical use case](#transactionsNoRetry) . 这意味着如果重试 `PROCESS` （5）块失败， `RETRY` （4）的行为如下：

抛出异常，在块级别回滚事务TX（2），并允许将项目重新呈现给输入队列 . 当项目重新出现时，可能会重新尝试，具体取决于适当的重试策略，再次执行PROCESS（5） . 第二次和后续尝试可能再次失败并重新抛出异常 . 最终，该项目最后一次重新出现 . 重试策略不允许另一次尝试，因此从不执行PROCESS（5） . 在这种情况下，我们遵循RECOVER（6）路径，有效地“跳过”已接收和正在处理的项目 . 

请注意，上面计划中用于 `RETRY` （4）的表示法明确表明输入步骤（4.1）是重试的一部分 . 它还清楚地表明有两个备用路径用于处理：正常情况，如 `PROCESS` （5）所示，以及恢复路径，如 `RECOVER` （6）在单独的块中所示 . 两条备用路径完全不同 . 在正常情况下只能使用一个 . 

在特殊情况下（例如特殊的 `TranscationValidException` 类型），重试策略可能能够确定 `RECOVER` （6）路径可以在 `PROCESS` （5）刚刚失败后的最后一次尝试中获取，而不是等待重新呈现该项目 . 这不是默认行为，因为它需要详细了解 `PROCESS` （5）块中发生的事情，这通常不可用 . 例如，如果输出在失败之前包含写访问，则应重新抛出异常以确保事务完整性 . 

外部 `REPEAT` （1）的完成政策对上述计划的成功至关重要 . 如果输出（5.1）失败，它可能抛出一个异常（它通常如所描述的那样），在这种情况下，事务 `TX` （2）失败，异常可以通过外部批处理 `REPEAT` （1）传播 . 我们不希望整个批次停止，因为如果我们再试一次， `RETRY` （4）可能仍然会成功，所以我们将 `exception=not critical` 添加到外部 `REPEAT` （1） . 

但请注意，如果 `TX` （2）失败并且我们再次尝试，则凭借外部完成策略，内部 `REPEAT` （3）中下一个处理的项目不能保证是刚刚失败的项目 . 它可能是，但它取决于输入的实现（4.1） . 因此，输出（5.1）可能会在新项目或旧项目上再次失败 . 批处理的客户端不应该假定每个 `RETRY` （4）尝试将处理与最后一个失败的项目相同的项目 . 例如，如果 `REPEAT` （1）的终止策略在10次尝试后失败，则在连续10次尝试后失败，但不一定在同一项目上 . 这与整体重试策略一致 . 内部 `RETRY` （4）知道每个项目的历史，并可以决定是否再次尝试它 . 


### A.4 . 异步块处理

通过将外部批处理配置为使用 `AsyncTaskExecutor` ，可以同时执行[typical example](#repeatRetry)中的内部批处理或块 . 外部批处理在完成之前等待所有块完成 . 以下示例显示了异步块处理：


```java
1   |  REPEAT(until=exhausted, concurrent, exception=not critical) {
|
2   |    TX {
3   |      REPEAT(size=5) {
|
4   |        RETRY(stateful, exception=deadlock loser) {
4.1 |          input;
5   |        } PROCESS {
|          output;
6   |        } RECOVER {
|          recover;
|        }
|
|      }
|    }
|
|  }
```



### A.5 . 异步项处理

原则上，[typical example](#repeatRetry)中块中的各个项目也可以同时处理 . 在这种情况下，事务边界必须移动到单个项的级别，以便每个事务都在单个线程上，如以下示例所示：


```java
1   |  REPEAT(until=exhausted, exception=not critical) {
|
2   |    REPEAT(size=5, concurrent) {
|
3   |      TX {
4   |        RETRY(stateful, exception=deadlock loser) {
4.1 |          input;
5   |        } PROCESS {
|          output;
6   |        } RECOVER {
|          recover;
|        }
|      }
|
|    }
|
|  }
```


该计划牺牲了简单计划所具有的优化效益，即将所有事务资源整合在一起 . 只有处理（5）的成本远高于交易管理成本（3）才有用 . 


### A.6 . 批处理和事务传播之间的交互

批量重试和事务管理之间的关联比我们理想的更紧密 . 特别是，无状态重试不能用于使用不支持NESTED传播的事务管理器重试数据库操作 . 

以下示例使用重试而不重复：


```java
1   |  TX {
|
1.1 |    input;
2.2 |    database access;
2   |    RETRY {
3   |      TX {
3.1 |        database access;
|      }
|    }
|
|  }
```


同样，出于同样的原因，即使 `RETRY` （2）最终成功，内部事务 `TX` （3）也会导致外部事务 `TX` （1）失败 . 

不幸的是，相同的效果从重试块渗透到周围的重复批次（如果有的话），如以下示例所示：


```java
1   |  TX {
|
2   |    REPEAT(size=5) {
2.1 |      input;
2.2 |      database access;
3   |      RETRY {
4   |        TX {
4.1 |          database access;
|        }
|      }
|    }
|
|  }
```


现在，如果TX（3）回滚，它可以污染TX（1）处的整个批次并强制它在结束时回滚 . 

那么非默认传播呢？


- 
在前面的示例中， `PROPAGATION_REQUIRES_NEW`  at  `TX` （3）防止外部 `TX` （1）在两个事务最终成功时被污染 . 但是如果 `TX` （3）提交并且 `TX` （1）回滚，那么 `TX` （3）将继续提交，因此我们违反了 `TX` （1）的交易 Contract  . 如果 `TX` （3）回滚， `TX` （1）不一定（但实际上可能会这样做，因为重试会引发回滚异常） . 


- 
 `PROPAGATION_NESTED`  at  `TX` （3）在重试的情况下（对于具有跳过的批处理）可以正常工作： `TX` （3）可以提交但随后由外部事务 `TX` （1）回滚 . 如果 `TX` （3）回滚， `TX` （1）在实践中回滚 . 此选项仅在某些平台上可用，不包括Hibernate或JTA，但它是唯一一个始终有效的平台 . 

因此，如果重试块包含任何数据库访问，则 `NESTED` 模式最佳 . 


### A.7 . 特例：正交资源交易

对于没有嵌套数据库事务的简单情况，默认传播总是正常的 . 请考虑以下示例，其中 `SESSION` 和 `TX` 不是全局 `XA` 资源，因此它们的资源是正交的：


```java
0   |  SESSION {
1   |    input;
2   |    RETRY {
3   |      TX {
3.1 |        database access;
|      }
|    }
|  }
```


这里有一个事务性消息 `SESSION` （0），但它不参与 `PlatformTransactionManager` 的其他事务，所以当 `TX` （3）启动时它不会传播 .   `RETRY` （2）块之外没有数据库访问权限 . 如果 `TX` （3）失败，然后最终成功重试， `SESSION` （0）即可提交（独立于 `TX` 块） . 这类似于vanilla "best-efforts-one-phase-commit"场景 . 当 `RETRY` （2）成功且 `SESSION` （0）无法提交（例如，因为消息系统不可用）时，可能发生的最坏情况是重复消息 . 


### A.8 . 无状态重试无法恢复

在上面的典型示例中，无状态重试和有状态重试之间的区别很重要 . 它实际上最终是一个强制区分的事务约束，这种约束也使得区分存在的原因显而易见 . 

我们首先观察到，除非我们在事务中包装项目处理，否则无法跳过失败并成功提交剩余块的项目 . 因此，我们将典型的批处理执行计划简化如下：


```java
0   |  REPEAT(until=exhausted) {
|
1   |    TX {
2   |      REPEAT(size=5) {
|
3   |        RETRY(stateless) {
4   |          TX {
4.1 |            input;
4.2 |            database access;
|          }
5   |        } RECOVER {
5.1 |          skip;
|        }
|
|      }
|    }
|
|  }
```


前面的示例显示了无状态 `RETRY` （3），其中 `RECOVER` （5）路径在最终尝试失败后启动 .   `stateless` 标签表示重复该块，而不会重新抛出任何异常，直至达到某个限制 . 这仅在事务 `TX` （4）具有NESTED传播时有效 . 

如果内部 `TX` （4）具有默认传播属性并回滚，则会污染外部 `TX` （1） . 事务管理器假定内部事务已损坏事务资源，因此不能再次使用它 . 

对NESTED传播的支持非常少见，我们选择不在当前版本的Spring Batch中支持无状态重试的恢复 . 通过使用上面的典型图案，总是可以实现相同的效果（以重复更多处理为代价） .