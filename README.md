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

## chapter 6-3
# Mockito를 이용한 테스트
테스트를 위한 class를 만들지 말고, Mockito를 이용한 테스트로 변경
```
   @Test
    public void mockUpgradeLevelsTest() {
        UserServiceImpl userServiceImpl = new UserServiceImpl();
        UserDao mockUserDao = mock(UserDao.class);
        when(mockUserDao.getAll()).thenReturn(this.users);

        userServiceImpl.setUserDao(mockUserDao);    //Mock Dao를 넣음

        MailSender mockMailSender = mock(MailSender.class);
        UserUpgradeImpl upgradePolicy = new UserUpgradeImpl();
        upgradePolicy.setMailSender(mockMailSender);

        userServiceImpl.setUpgradePolicy(upgradePolicy);

        userServiceImpl.upgradeLevels();

        verify(mockUserDao, times(2)).update(any(User.class));
        verify(mockUserDao, times(2)).update(any(User.class));
        verify(mockUserDao).update(users.get(1));
        Assertions.assertTrue(users.get(1).getLevel() == Level.SILVER);
        verify(mockUserDao).update(users.get(3));
        Assertions.assertTrue(users.get(3).getLevel() == Level.GOLD);

    }
```
1.  mock(`className`); 해당 클래스에 대한 Mock을 생성
2.  when(`mockObject.method`).thenReturn(`return value`); Mock객체의 메서드가 호출될 때, 반환될 값을 설정
3.  verify(`mockObject`,times(`expected times`)).upate(any(`type of return value`));  
반환값의 종류에 상관없이(`any`), upate(`target Mehtod`)가 몇 번 호출되었는지 설정

## chapter 6-4
# 다이나믹 프록시를 이용한 트랜잭션
1.  InvocationHandler을 구현한 클래스
```
public class TransactionHandler implements InvocationHandler {

    private Object target;
    private PlatformTransactionManager transactionManager;
    private String pattern;

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(
        PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        this.pattern = pattern;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        if (method.getName().startsWith(pattern)) {
            return invokeInTransaction(method, args);
        }
        return method.invoke(method, args);
    }

    private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
        TransactionStatus status = this.transactionManager
            .getTransaction(new DefaultTransactionDefinition());

        try {
            Object ret = method.invoke(target, args);
            this.transactionManager.commit(status);
            return ret;
        } catch (InvocationTargetException e) {
            this.transactionManager.rollback(status);
            throw e.getTargetException();
        }
    }
}
```
2.  테스트 코드 변경
(기존의 UserServiceImpl는 트랙잭션이 적용되지 않았다.)
```
 @Test
    public void mockUpgradeLevelsWithTransactionHandlerTest() {

        UserServiceImpl UserService = new UserServiceImpl();

        UserDao mockUserDao = mock(UserDao.class);
        when(mockUserDao.getAll()).thenReturn(this.users);

        MailSender mockMailSender = mock(MailSender.class);
        UserUpgradeImpl upgradePolicy = new UserUpgradeImpl();
        upgradePolicy.setMailSender(mockMailSender);

        UserService.setUserDao(mockUserDao);
        UserService.setUpgradePolicy(upgradePolicy);

        TransactionHandler transactionHandler = new TransactionHandler();
        transactionHandler.setPattern("upgradeLevels");
        transactionHandler.setTarget(UserService);
        transactionHandler.setTransactionManager(this.transactionManager);

        UserService txUserService = (UserService) Proxy.newProxyInstance(
            getClass().getClassLoader()
            , new Class[]{UserService.class}
            , transactionHandler);
        txUserService.upgradeLevels();
		//... 
    }
```

## chapter 6-6
# 다이나믹 프록시 Bean으로 등록
1.  스프링은 Bean을 생성하는 방법을 생성자 방법뿐만 아니라, FactoryBean인터페이스도 이용할 수 있다.
```
public class TxProxyFactoryBean implements FactoryBean<Object> {

    private Object target;
    private PlatformTransactionManager transactionManager;
    private String pattern;
    private Class<?> serviceInterface;

    public void setServiceInterface(Class<?> serviceInterface) {
        this.serviceInterface = serviceInterface;
    }

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(
        PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        this.pattern = pattern;
    }

    @Override
    public Object getObject() throws Exception {
        TransactionHandler txHandler = new TransactionHandler();
        txHandler.setTarget(target);
        txHandler.setTransactionManager(transactionManager);
        txHandler.setPattern(pattern);
        return Proxy.newProxyInstance(
            getClass().getClassLoader()
            , new Class[]{serviceInterface}
            , txHandler);
    }

    @Override
    public Class<?> getObjectType() {
        return serviceInterface;
    }

    @Override
    public boolean isSingleton() {
        return false;
    }
}
```
2.  xml의 userService 변경
```
  <bean id="userService" class="springbook.user.service.`TxProxyFactoryBean`">
    <property name="target" ref="userServiceImpl"/>
    <property name="pattern" value="upgradeLevels"/>
    <property name="transactionManager" ref="transactionManager"/>
    <property name="serviceInterface" value="springbook.user.service.UserService"/>
  </bean>
```
이와 같은 방식으로 다른 객체에도 트랜잭션을 적용할 수 있다.

3.  Test 변경
```
    @Test
    @DirtiesContext
    public void upgradeAllOrNothing() throws Exception {
        userDao.deleteAll();

        UserServiceImpl userService = new UserServiceImpl();
        UserUpgradeImpl upgradePolicy = new UserUpgradeImpl();
        upgradePolicy.setMailSender(mock(MailSender.class));

        userService.setUpgradePolicy(new TestUserUpgradePolicy(users.get(3).getId()));
        userService.setUserDao(this.userDao);

        TxProxyFactoryBean txProxyFactoryBean
            = context.getBean("&userService", TxProxyFactoryBean.class);

        txProxyFactoryBean.setTarget(userService);
        UserService txUserService = (UserService) txProxyFactoryBean.getObject();
        for (User user : users) {
            txUserService.add(user);
        }
        try {
            txUserService.upgradeLevels();
            fail("TestUserServiceException expected");
        } catch (TestUserServiceException e) {
            checkLevelUpgraded(users.get(1), false);
        }
    }
```

## chapter 6-6
# ProxyFactoryBean으로 다이나믹 프록시 등록할
1.  InvocationHandler는 Target을 설정해야 했지만 ProxyFactoryBean은 설정할 필요가 없다.
2.  setInstance를 통해서 반환 타입을 정해줘야 했지만, ProxyFactoryBean은 알아서 설정해준다.
3.  ProxyFactoryBean는 적용될 메서드를 직접 설정할 수 있다.
```
ProxyFactoryBean pfBean = new ProxyFactoryBean();

        pfBean.setTarget(new HelloTarget());

        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        pointcut.addMethodName("*Hi");
        pointcut.addMethodName("*Hello");
        pointcut.addMethodName("*ThankYou");
        
        pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut,new UppercaseAdvice()));

        Hello hello = (Hello) pfBean.getObject();
```
4.  어드바이저 = 포인트컷(메소드 선정 알고리즘)+ 어드바이스(부가기능)
	-  addAdvisor()는 부가기능(advice)와 포인트컷(methodName)을 동시에 받을 수 있다.
	-  addAdvice()는 부가기능(advice)만 받는다.
	-  NameMatchMethodPointcut은 내부적으로 메서드 이름을 List형태로 저장한다.
```
public class NameMatchMethodPointcut extends StaticMethodMatcherPointcut implements Serializable {
    private List<String> mappedNames = new LinkedList();
	...
	    public NameMatchMethodPointcut addMethodName(String name) {
        this.mappedNames.add(name);
        return this;
    }
	...
	}
```
5.  트랜잭션이 로직이 설정된 TransactionAdvice
```
public class TransactionAdvice implements MethodInterceptor {

    private PlatformTransactionManager transactionManager;

    public void setTransactionManager(
        PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        TransactionStatus status = this.transactionManager
            .getTransaction(new DefaultTransactionDefinition());
        try {
            Object ret = invocation.proceed();
            this.transactionManager.commit(status);
            return ret;
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```
	-  재사용이 가능하다.
	-  target을 따로 받지 않아서, 다른 객체에 트랜잭션을 적용할 때도 재사용이 가능하다.
6.  xml파일 설정
```
  <bean id="transactionAdvice" class="springbook.user.service.TransactionAdvice">
    <property name="transactionManager" ref="transactionManager"/>
  </bean>

  <bean id="transactionPointcut" class="org.springframework.aop.support.NameMatchMethodPointcut">
    <property name="mappedName" value="upgrade*"/>
  </bean>

  <bean id="transactionAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
    <property name="advice" ref="transactionAdvice"/>
    <property name="pointcut" ref="transactionPointcut"/>
  </bean>

  <bean id="userService" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target" ref="userServiceImpl"/>
    <property name="interceptorNames">
      <list>
        <value>transactionAdvisor</value>
        <!--한 개 이상의 부가기능을 넣을 수 있다.-->
      </list>
    </property>
  </bean>
```
	-  userService는 이제 ProxyFactoryBean
	-  String[]으로 적용될 DefaultPointcutAdvisor를 구현한 객체를 저장한다.(DefaultPointcutAdvisor에 적용될 메서드 이름과 부기가능(Advisor구현체)가 들어있다.)
```
public class ProxyFactoryBean extends ProxyCreatorSupport implements FactoryBean<Object>, BeanClassLoaderAware, BeanFactoryAware {
	...
    private String[] interceptorNames;
	...
	}
```	


