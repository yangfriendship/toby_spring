# toby_spring
토비의스프링 내용을 정리한 저장소입니다.


## chapter 5-2
토비의 스프링 vol.1 ~369P

# 트랙잭션 문제

1. Transaction을 사용하려면 Connection에서 setAutoCommit을 false로 지정해야한다.
2. 하지만 이전 코드에서는 UserDao에서 Connection을 주입받아 사용했다.
3. `로직을 실행하고 트랜잭션이 필요한 구간`(UserService)과 `Connection을 주입받아 트랜잭션을 할 수 있는 구간`(UserDao)이 서로 다르다
4. UserService에서 Connection을 생성한 후, UserDao에 Connection을 주입하는 방법(x) -> Connection을 이리저리 주고 받고 코드가 더러워진다.
5. upgradeLevels 메서드를 UserDao로 이동시킴(X) -> Data Access를 담당하는 UserDao가 쓸데없는 책임을 갖게 된다.

### 해결 방안
``` 
    public void upgradeLevels() throws SQLException {
        TransactionSynchronizationManager.initSynchronization();
        Connection connection = DataSourceUtils.getConnection(dataSource);
        connection.setAutoCommit(false);

        try {
            List<User> users = userDao.getAll();
            for (User user : users) {
                if (upgradePolicy.canUpgradeLevel(user)) {
                    upgradePolicy.upgradeLevel(user, userDao);
                }
            }
            connection.commit();
        } catch (Exception e) {
            connection.rollback();
            throw e;
        } finally {
            DataSourceUtils.releaseConnection(connection, dataSource);
            TransactionSynchronizationManager.unbindResource(this.dataSource);
            TransactionSynchronizationManager.clearSynchronization();
        }
    }

```
Spring에서 제공하는 TransactionSynchronizationManager를 이용
1.  Connection을 파라미터로 보낼 필요가 없다.
2.  Dao를 사용하는 Service단에서 Transaction을 사용할 수 있다.


## chapter 5-3

1.  로컬 트랜잭션은 하나의 DB Connection에 종속된다.
2.  각 DB와 독립적으로 만들어지는 Connection을 통해서가 아니라， 별도의 트랜잭션 관리자를 통해 트랜잭션을 관리히는 글로벌 트랜잭션global tran ction 방식을 사용해야 한다.
3.  JDBC 외에 이런 글로벌 트랜잭션을 지원히는 트랜잭션 매니저를 지원하기 위한 API 인 JTAJava Transaction API를 제공한다.
4.  하나 이상의 DB가 참여히는 트랜잭션을 만들려면 JTA를 사용해야 한다는 사실만 기억해두자.

### Spring의 PlatformTransactionManager 적용
```
public void upgradeLevels() throws SQLException {
        PlatformTransactionManager transactionManager = new DataSourceTransactionManager(
            dataSource);

        // 트랜잭션 시작
        TransactionStatus status = transactionManager
            .getTransaction(new DefaultTransactionDefinition());

        try {
            List<User> users = userDao.getAll();
            for (User user : users) {
                if (upgradePolicy.canUpgradeLevel(user)) {
                    upgradePolicy.upgradeLevel(user, userDao);
                }
            }
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
```

## chapter 5-4
1.  PlatformTransactionManager을 Spring Bean으로 등록
```
  private PlatformTransactionManager transactionManager;

    public void setTransactionManager(
        PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }
```

```
  <bean id="userService" class="springbook.user.service.UserService">
    <property name="userDao" ref="userDao"/>
    <property name="upgradePolicy" ref="userLevelUpgradePolicy"/>
    <property name="transactionManager" ref="transactionManager"/>
  </bean>
  //...
    <bean id="transactionManager"
    class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
  </bean>
```
-  관례상 PlatformTransactionManager을 transactionManager라고 사용한다.
-  Spring에 Bean을 등록할 때는 싱글톤에서 안전한지 확인해야한다. PlatformTransactionManager은 스프링에서 지원하는 기능이므로, Bean으로 등록해도 상관없다.
-  JTA를 사용해야하는 경우에는 DefaultTransactionDefinition 대신 JtaTransaction Manager을 사용하면 된다.
	```
	<bean id="transactionManager" 
	class="org .springframework.transaction . jta. JtaTransaction Manager" />
	//JtaTransactionManager는 애플리케이션 서버의 트랜잭션 서비스를 이용하기 때문에 직접 DataSource와 연동할 필요는 없다.
	```




