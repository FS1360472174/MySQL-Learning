
1. 连接池管理

与DataSourceUtils的区别

2. 引入 Spring-jdbc
进行transaction 管理

```
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <tx:annotation-driven transaction-manager="transactionManager"/>
```
3. 这时候连接池管理交给了SpringManagedTransaction

org.mybatis.spring.transaction.SpringManagedTransaction

```
    private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
        Connection connection = this.getConnection(statementLog);
        Statement stmt = handler.prepare(connection, this.transaction.getTimeout());
        handler.parameterize(stmt);
        return stmt;
    }

```

4. TDDL

在执行的时候TPreparedStatment,
重新包装了SQL，PreparedStatment
AutoCommitTransaction管理一个连接
getConnection,这时候Connection
获取的是TGroupConnection中createNewConnection

ResultSet使用的是DruidPooledResultSet


SqlSessionUtils.closeSqlSession

TConnection.close()
TConnectionWrapper.close()
DruidPooledConnection.close()
DruidPooledConnection.syncClose()
DruidPooledConnection.recycle()

```
public DruidPooledConnection getConnectionDirect(long maxWaitMillis) throws SQLException {
        int notFullTimeoutRetryCnt = 0;

        DruidPooledConnection poolableConnection;
        ...
        if(this.isRemoveAbandoned()) {
            StackTraceElement[] stackTrace = Thread.currentThread().getStackTrace();
            poolableConnection.setConnectStackTrace(stackTrace);
            poolableConnection.setConnectedTimeNano();
            poolableConnection.setTraceEnable(true);
            Map var21 = this.activeConnections;
            synchronized(this.activeConnections) {
                // 将一个活跃connection
                this.activeConnections.put(poolableConnection, PRESENT);
            }
        }

        if(!this.isDefaultAutoCommit()) {
            poolableConnection.setAutoCommit(false);
        }

        return poolableConnection;
    }

```


remove abon

```
  public int removeAbandoned() {
        int removeCount = 0;
        long currrentNanos = System.nanoTime();
        List<DruidPooledConnection> abandonedList = new ArrayList();
        Map var5 = this.activeConnections;
        synchronized(this.activeConnections) {
            // 获取这边的值
            Iterator iter = this.activeConnections.keySet().iterator();

            while(iter.hasNext()) {
                DruidPooledConnection pooledConnection = (DruidPooledConnection)iter.next();
                if(!pooledConnection.isRunning()) {
                    long timeMillis = (currrentNanos - pooledConnection.getConnectedTimeNano()) / 1000000L;
                    if(timeMillis >= this.removeAbandonedTimeoutMillis) {
                        iter.remove();
                        pooledConnection.setTraceEnable(false);
                        abandonedList.add(pooledConnection);
                    }
                }
            }
        }

        if(abandonedList.size() > 0) {
            Iterator var14 = abandonedList.iterator();
            while(true) {
                DruidPooledConnection pooledConnection;
                do {
                    while(true) {
                        if(!var14.hasNext()) {
                            return removeCount;
                        }

                        pooledConnection = (DruidPooledConnection)var14.next();
                        synchronized(pooledConnection) {
                            if(!pooledConnection.isDisable()) {
                                break;
                            }
                        }
                    }

                    JdbcUtils.close(pooledConnection);
                    pooledConnection.abandond();
                    ++this.removeAbandonedCount;
                    ++removeCount;
                } while(!this.isLogAbandoned());

                StringBuilder buf = new StringBuilder();
                buf.append("abandon connection, owner thread: ");
                LOG.error(buf.toString());
            }
        } else {
            return removeCount;
        }
    }


```
# 参考#

http://blog.csdn.net/luanlouis/article/details/40422941

http://blog.csdn.net/luanlouis/article/details/37671851


