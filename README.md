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
1. Connection을 파라미터로 보낼 필요가 없다.
2. Dao를 사용하는 Service단에서 Transaction을 사용할 수 있다.


## chapter 5-4

