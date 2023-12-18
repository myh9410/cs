> HikariDatasource와 HikariCP의 동작 방식을 이해해보자
> 
> springboot >= 2.7 / jdk >= 17

### HikariDataSource Configuration
특정 DB에 대한 DataSource 설정 시 일반적으로 HikariDataSource를 많이 사용하고, 방식은 회사, 개발자마다 다르겠지만 기본적인 틀은 아래와 같다.

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;

//JPA EntityManagerFactory 설정
@Bean(name = "someEntityManager")
public LocalContainerEntityManagerFactoryBean someEntityManager(
@Qualifier("someDataSource") DataSource someDataSource
) {
    return new EntityManagerFactoryBuilder()
        .dataSource(someDataSource)
        .packages(...)
        .persistenceUnit(...)
        .build();
}

//Hikari 기반 datasource 설정
@Bean(name = "someDataSource")
public DataSource someDataSource(){
        HikariDataSource dataSource=new HikariDataSource();
        datasource.setDriverClassName(...);
        datasource.setJdbcUrl(...);
        datasource.setUsername(...);
        datasource.setPassword(...);
        ...

        return datasource;
        }
```

### HikariDataSource구성과 HikariDataSource.getConnection()의 동작 방식
DB 접근 시도 시, 위의 DataSource 설정값을 통해 DB와의 connection을 맺는다. - HikariDataSource.getConnection()  
HikariDataSource는 connection 요청 시, HikariPool을 생성하고 HikariPoll을 통해 connection을 관리한다.

```java
public class HikariDataSource extends HikariConfig implements DataSource, Closeable {
    /* volatile 이란? 
       - Main Memory를 통해 변수의 값을 관리한다. 
       - 읽기 쓰기 시, Multi Thread 환경에서 각 Thread가 참조하는 cache의 공통 변수 값이 불일치할 수 있는 문제를 방지한다.
       - main memory까지 접근하기 때문에 상대적으로 변수 접근 시 속도가 느리지만, concurrent read, write가 발생하는 Multi Thread환경에서 Thread-Safe를 보장한다. 
     */
    
    private final HikariPool fastPathPool; //pool이 이미 설정되어서 사용 가능한 경우, 불필요한 synchronized block 동작 없이 즉시 접근을 위한 변수
    private volatile HikariPool pool; //fastPathPool 정보가 없는 경우, volatile한 pool변수를 통해 synchronized block 내 connection 시도를 위한 변수
    
    ...

    public HikariDataSource(HikariConfig configuration)
    {
        ...
        
        LOGGER.info("{} - Starting...", configuration.getPoolName());
        pool = fastPathPool = new HikariPool(this);
        LOGGER.info("{} - Start completed.", configuration.getPoolName());

        this.seal();
    }

    @Override
    public Connection getConnection() throws SQLException
    {
        ...
        
        if (fastPathPool != null) {
            return fastPathPool.getConnection();
        }

        // See http://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java
        HikariPool result = pool;
        if (result == null) {
            synchronized (this) {
                result = pool;
                if (result == null) {
                    validate();
                    LOGGER.info("{} - Starting...", getPoolName());
                    try {
                        pool = result = new HikariPool(this);
                        this.seal();
                    }
                    catch (PoolInitializationException pie) {
                        if (pie.getCause() instanceof SQLException) {
                            throw (SQLException) pie.getCause();
                        }
                        else {
                            throw pie;
                        }
                    }
                    LOGGER.info("{} - Start completed.", getPoolName());
                }
            }
        }

        return result.getConnection();
    }
}
```

### HikariPool구성과 HikariPool.getConnection()의 동작 방식
위 HikariDataSource.getConnection() 함수 내에서 HikariPool의 getConnection() 함수를 호출하는 것을 확인할 수 있다.  
HikariPool은 여러 정보를 갖고 있지만, ConcurrentBag<PoolEntry>와 getConnection(final long hardTimeout) 함수만 보자.

```java
public final class HikariPool extends PoolBase implements HikariPoolMXBean, IBagStateListener
{
    ...

    /*
        PoolEntry란?
        - Connection 정보, 상태(boolean isReadOnly, boolean isAutoCommit 등), 접근 시간 등의 정보들을 갖는 객체
     */
    private final ConcurrentBag<PoolEntry> connectionBag;
    private final SuspendResumeLock suspendResumeLock;
    
    ...

    public Connection getConnection(final long hardTimeout) throws SQLException
    {
        suspendResumeLock.acquire();
        final long startTime = currentTime();

        try {
            long timeout = hardTimeout;
            do {
                PoolEntry poolEntry = connectionBag.borrow(timeout, MILLISECONDS);
                if (poolEntry == null) {
                    break; // We timed out... break and throw exception
                }

                final long now = currentTime();
                if (poolEntry.isMarkedEvicted() || (elapsedMillis(poolEntry.lastAccessed, now) > aliveBypassWindowMs && !isConnectionAlive(poolEntry.connection))) {
                    closeConnection(poolEntry, poolEntry.isMarkedEvicted() ? EVICTED_CONNECTION_MESSAGE : DEAD_CONNECTION_MESSAGE);
                    timeout = hardTimeout - elapsedMillis(startTime);
                }
                else {
                    metricsTracker.recordBorrowStats(poolEntry, startTime);
                    return poolEntry.createProxyConnection(leakTaskFactory.schedule(poolEntry), now);
                }
            } while (timeout > 0L);

            metricsTracker.recordBorrowTimeoutStats(startTime);
            throw createTimeoutException(startTime);
        }
        catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new SQLException(poolName + " - Interrupted during connection acquisition", e);
        }
        finally {
            suspendResumeLock.release();
        }
    }
}
```

connectionBag.borrow()를 통해 PoolEntry 정보를 조회한다.  
그렇다면, PoolEntry들이 저장되는 ConcurrentBag란 무엇일까?

### ConcurrentBag구성과 SynchronousQueue

```java
public class ConcurrentBag<T extends IConcurrentBagEntry> implements AutoCloseable
{
    ...
    private final AtomicInteger waiters;
    private final CopyOnWriteArrayList<T> sharedList;
    private final ThreadLocal<List<Object>> threadList;
    private final SynchronousQueue<T> handoffQueue;

    /*
        borrow 시 사용 가능한 PoolEntry 조회 순서
        1. threadList - 현재 Thread에서 이전에 사용된 PoolEntry 정보를 갖는다. threadList가 있다면, 해당 PoolEntry를 재사용하여 Connection을 맺는다.
        2. sharedList - 전체 Thread에서 사용 가능한 전체 PoolEntry 정보를 갖는다. threadList가 없다면, sharedList에 저장된 PoolEntry를 통해 Connection을 맺는다.
        3. handoffQueue - Thread들이 Connection 사용을 위해 대기하고 있는 queue. Connection 사용 후 반환 시, queue에 대기하고 있는 (waiters) Thread가 있다면 해당 queue에 PoolEntry를 offer한다.
     */
    public T borrow(long timeout, final TimeUnit timeUnit) throws InterruptedException
    {
        // Try the thread-local list first
        final List<Object> list = threadList.get();
        for (int i = list.size() - 1; i >= 0; i--) {
            final Object entry = list.remove(i);
            @SuppressWarnings("unchecked")
            final T bagEntry = weakThreadLocals ? ((WeakReference<T>) entry).get() : (T) entry;
            if (bagEntry != null && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
                return bagEntry;
            }
        }

        // Otherwise, scan the shared list ... then poll the handoff queue
        final int waiting = waiters.incrementAndGet();
        try {
            for (T bagEntry : sharedList) {
                if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
                    // If we may have stolen another waiter's connection, request another bag add.
                    if (waiting > 1) {
                        listener.addBagItem(waiting - 1);
                    }
                    return bagEntry;
                }
            }

            listener.addBagItem(waiting);

            timeout = timeUnit.toNanos(timeout);
            do {
                final long start = currentTime();
                final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
                if (bagEntry == null || bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
                    return bagEntry;
                }

                timeout -= elapsedNanos(start);
            } while (timeout > 10_000);

            return null;
        }
        finally {
            waiters.decrementAndGet();
        }
    }
}
```

ConnectionBag이 갖는 정보 ( ThreadList, SharedList, HandoffQueue )
>   - ThreadList - ThreadLocal<List<Object>>
>     - ThreadLocal의 이전에 연결된 정보를 저장하기 위한 List
>   - SharedList - CopyOnWriteArrayList(thread safe한 array)
>     - 전체 Thread에서 사용할 수 있는 Connection Pool의 모든 Connection 정보를 갖는 리스트
>   - HandoffQueue - SynchronousQueue
>     - put과 take를 통해 block 처리 되는 queue : put되면, take 전까지 block 되고 그 반대도 동일한 형식 
>     - 반환되는 커넥션 사용을 위한 대기 큐