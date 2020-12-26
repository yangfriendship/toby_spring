# toby_spring
토비의스프링 내용을 정리한 저장소입니다.


## chapter 5-3
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


## chapter 5-4

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

## chapter 5-5
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

## chapter 5-6
Mail보내는 기능 생성(Dummy)
1.  LevelUpgrade 로직을 따로 인터페이스를 생성해 분리했다. 책과 구조가 조금 다르다.

```
//  springbook.user.service.UserUpgradeImpl;

    @Override
    public void upgradeLevel(User user, final UserDao userDao) {
        user.upgradeLevel();
        userDao.update(user);
        sendUpgradeEmail(user);
    }


    private void sendUpgradeEmail(User user) {
        SimpleMailMessage mailMessage = new SimpleMailMessage();

        mailMessage.setTo(user.getEmail());
        mailMessage.setFrom("seradmin@ksug.org");
        mailMessage.setSubject("Upgrade 안내 ");
        mailMessage.setText("사용자님의 등급이 " + user.getLevel().name());
        mailSender.send(mailMessage);
    }
```
2.  Bean으로 등록

```
  <bean id="userLevelUpgradePolicy" class="springbook.user.service.UserUpgradeImpl">
    <property name="mailSender" ref="mainSender"/>
  </bean>

  <bean id="mainSender" class="springbook.user.service.DummyMailSender"/>

```
## chapter 5-7
Mock을 이용한 EmailSend Test(DB와 User에 Email을 추가했으니, Dao에 Email을 추가하도록 수정해야한다.)

1. MailSender을 구현한 MockMailSender
```
public class MockMailSender implements MailSender {

    private List<String> requests = new ArrayList<>();

    public List<String> getRequests() {
        return requests;
    }

    @Override
    public void send(SimpleMailMessage simpleMailMessage) throws MailException {
        requests.add(simpleMailMessage.getTo()[0]);
    }

    @Override
    public void send(SimpleMailMessage[] simpleMailMessages) throws MailException {
    }
    
```
2. MockMailSender를 이용한 Test
```
	@Test
    @DirtiesContext	// 컨텍스트의 DI 설정올 변경하는 테스트라는 것을 알려준다.
    public void upgradeLevels() {
        userDao.deleteAll();

        MockMailSender mockMailSender = new MockMailSender();
        UserUpgradeImpl userUpgrade = new UserUpgradeImpl();
        userUpgrade.setMailSender(mockMailSender);

        userService.setUpgradePolicy(userUpgrade);

        for (User user : users) {
            userService.add(user);
        }

        userService.upgradeLevels();

        checkLevelUpgraded(users.get(0), false);
        checkLevelUpgraded(users.get(1), true);
        checkLevelUpgraded(users.get(2), false);
        checkLevelUpgraded(users.get(3), true);
        checkLevelUpgraded(users.get(4), false);

        List<String> requests = mockMailSender.getRequests();

        Assertions.assertTrue(requests.size() == 2);
        Assertions.assertTrue(requests.get(0).equals(users.get(1).getEmail()));
        Assertions.assertTrue(requests.get(1).equals(users.get(3).getEmail()));

    }
```

## chapter 6-1

### Transaction과 UserService 분리
1.  UserService에서 Transaction을 분리
2.  UserService를 Interaface로 선언
```
public interface UserService {

    void add(User user);

    void upgradeLevels();

}
```
3.  기존의 UserService를 `UserServiceImpl`로 변경(Transaction관련된 코드 모두 제거)
4.  Transaction기능이 추가된 `UserServiceTx`를 생성( `UserServiceImpl`를 주입받는다)
```
public class UserServiceTx implements UserService {

    private UserService userService;
    private PlatformTransactionManager transactionManager;

    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    public void setTransactionManager(
        PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    @Override
    public void add(User user) {
        userService.add(user);
    }

    @Override
    public void upgradeLevels() {

        TransactionStatus status = transactionManager
            .getTransaction(new DefaultTransactionDefinition());

        try {

            userService.upgradeLevels();
            this.transactionManager.commit(status);
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }

    }
}
```
5.  xml파일 수정
```
  <bean id="userService" class="springbook.user.service.UserServiceTx">
    <property name="userService" ref="userServiceImpl"/>
    <property name="transactionManager" ref="transactionManager" />
  </bean>

  <bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
    <property name="userDao" ref="userDao"/>
    <property name="upgradePolicy" ref="userLevelUpgradePolicy"/>
  </bean>
```
이로써 Dao를 이용하여 비즈니스 로직을 수행하는 Service와 Transaction기능을 분리했다.

6.  TestCode 수정
```
   @Test
    public void upgradeAllOrNothing() {
        userDao.deleteAll();

        TestUserUpgradePolicy testUserUpgradePolicy = new TestUserUpgradePolicy(
            users.get(3).getId());
        userServiceImpl.setUpgradePolicy(testUserUpgradePolicy);
        userServiceImpl.setUserDao(userDao);

        UserServiceTx userServiceTx = new UserServiceTx();
        userServiceTx.setUserService(userServiceImpl);
        userServiceTx.setTransactionManager(transactionManager);

        for (User user : users) {
            userServiceTx.add(user);
        }

        try {
            userServiceTx.upgradeLevels();
            fail("TestUserServiceException Expected");
        } catch (TestUserServiceException e) {
            // catch로 꼭 넘어와야 한다.
        }
        checkLevelUpgraded(users.get(1), false);
    }
```

## chapter 6-2
### MockUserDao를 이용한 테스트

1.  MockUserDao 생성
```
public class MockUserDao implements UserDao {

    private List<User> users;
    private List<User> updated = new ArrayList<>();

    public MockUserDao(List<User> users) {
        this.users = users;
    }

    public List<User> getUpdated() {
        return updated;
    }

    @Override
    public List<User> getAll() {
        return this.users;
    }



    @Override
    public int update(User user) {
        updated.add(user);
        return 0;
    }
	
	// 사용하지 않는 나머지 메서드들은 throw new UnsupportedOperationException()을 사용해서 해당 메서드를
	// 사용한다면 예외를 던져주도록 설정한다.
	@Override
    public void deleteAll() {
        throw new UnsupportedOperationException();
    }
	...
}
```
2.  Test 수정
```
  @Test
    @DirtiesContext
    public void upgradeLevels() {
        userDao.deleteAll();

        MockMailSender mockMailSender = new MockMailSender();
        UserUpgradeImpl userUpgrade = new UserUpgradeImpl();
        userUpgrade.setMailSender(mockMailSender);

        MockUserDao mockUserDao = new MockUserDao(this.users);

        userServiceImpl.setUserDao(mockUserDao);
        userServiceImpl.setUpgradePolicy(userUpgrade);

        userService.upgradeLevels();

        List<User> updated = mockUserDao.getUpdated();

        Assertions.assertTrue(updated.size() == 2);

        checkUserAndLevel(updated.get(0), users.get(1).getId(), Level.SILVER);
        checkUserAndLevel(updated.get(1), users.get(3).getId(), Level.GOLD);

        List<String> requests = mockMailSender.getRequests();

        Assertions.assertTrue(requests.size() == 2);
        Assertions.assertTrue(requests.get(0).equals(users.get(1).getEmail()));
        Assertions.assertTrue(requests.get(1).equals(users.get(3).getEmail()));

    }
```

