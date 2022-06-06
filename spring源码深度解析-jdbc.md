# 数据库连接JDBC

## 示例展示

```
@Data
@AllArgsConstructor
@ToString
public class User {

    private int id;
    private String name;
    private int age;
    private String sex;


    public User() {

    }
}
```

```
public class UserRowMapper implements RowMapper {

    public Object mapRow(ResultSet rs, int rowNum) throws SQLException {
        User person=new User(rs.getInt("id"),
                rs.getString("name"),
                rs.getInt("age"),
                rs.getString("sex"));
        return person;
    }

}
```

```
public interface UserService {

    void save(User user);

    List<User> getUsers();

}
```

```
public class UserServiceImpl implements UserService {

    private JdbcTemplate JdbcTemplate;

    public void setDataSource(DataSource dataSource) {
        this.JdbcTemplate=new JdbcTemplate(dataSource);
    }

    public void save(User user) {
        JdbcTemplate.update("insert into user(name,age,sex) values(?,?,?)",
                new Object[]{user.getName(), user.getAge(), user.getSex()}, new int[]{
                        Types.VARCHAR,
                        Types.INTEGER,
                        Types.VARCHAR,
                });
    }

    public List<User> getUsers() {
        List<User> list = JdbcTemplate.query("select * from user", new UserRowMapper());
        return list;
    }
}
```

```
	<bean id="dataSource"
		  class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
		<property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
		<property name="url" value="jdbc:mysql://192.168.152.5:3306/biyi_test?allowPublicKeyRetrieval=true"/>
		<property name="username" value="root"/>
		<property name="password" value="v3imYJ2@yL6Aq6Tu"/>
		<property name="initialSize" value="1" />
	</bean>

	<bean id="userService" class="com.zchuber.springsourcedeepdiving.jdbc.UserServiceImpl" >
		<property name="dataSource" ref="dataSource"/>
	</bean>
```

测试用例：

```
    @Test
    public void testSimpleLoad(){
        ApplicationContext ac=new ClassPathXmlApplicationContext("beanFactoryTest-jdbc.xml");

        UserService userService=(UserService)ac.getBean("userService");
        User user = new User();
        user.setName("张三");
        user.setAge(20);
        user.setSex("man");

        userService.save(user);

        List<User> personList = userService.getUsers();
        for (User u : personList) {
            System.out.println(u);
        }
    }
```

运行结果：

```
User(id=1, name=张三, age=20, sex=man)
```

## JdbcTemplate的query原理

JdbcTemplate#update：

```
	public int update(String sql, Object[] args, int[] argTypes) throws DataAccessException {
		//将args和对应的类型封装为ArgumentTypePreparedStatmentSetter
		return update(sql, newArgTypePreparedStatementSetter(args, argTypes));
	}
```

JdbcTemplate#update：

```
	public int update(String sql, @Nullable PreparedStatementSetter pss) throws DataAccessException {
		//将sql封装为SimplePreparedStatmentCreator
		return update(new SimplePreparedStatementCreator(sql), pss);
	}
```

JdbcTemplate#update：

```
	protected int update(final PreparedStatementCreator psc, @Nullable final PreparedStatementSetter pss)
			throws DataAccessException {

		logger.debug("Executing prepared SQL update");
		//调用jdbctemplate的execute方法
		return updateCount(execute(psc, ps -> {
			try {
				//为preparedStatment设置值
				if (pss != null) {
					pss.setValues(ps);
				}
				int rows = ps.executeUpdate();
				if (logger.isTraceEnabled()) {
					logger.trace("SQL update affected " + rows + " rows");
				}
				return rows;
			}
			finally {
				if (pss instanceof ParameterDisposer) {
					((ParameterDisposer) pss).cleanupParameters();
				}
			}
		}, true));
	}
```

JdbcTemplate#execute：

```
	private <T> T execute(PreparedStatementCreator psc, PreparedStatementCallback<T> action, boolean closeResources)
			throws DataAccessException {
		
		//获得数据库连接，关键方法
		Connection con = DataSourceUtils.getConnection(obtainDataSource());
		PreparedStatement ps = null;
		try {
			//创建preparedStatment
			ps = psc.createPreparedStatement(con);
			//设置ps
			applyStatementSettings(ps);
			//执行ps
			T result = action.doInPreparedStatement(ps);
			//处理警告
			handleWarnings(ps);
			return result;
		}
		catch (SQLException ex) {
			// Release Connection early, to avoid potential connection pool deadlock
			// in the case when the exception translator hasn't been initialized yet.
			if (psc instanceof ParameterDisposer) {
				((ParameterDisposer) psc).cleanupParameters();
			}
			String sql = getSql(psc);
			psc = null;
			JdbcUtils.closeStatement(ps);
			ps = null;
			DataSourceUtils.releaseConnection(con, getDataSource());
			con = null;
			throw translateException("PreparedStatementCallback", sql, ex);
		}
		finally {
			if (closeResources) {
				if (psc instanceof ParameterDisposer) {
					((ParameterDisposer) psc).cleanupParameters();
				}
				JdbcUtils.closeStatement(ps);
				DataSourceUtils.releaseConnection(con, getDataSource());
			}
		}
	}
```

#### DataSourceUtils#getConnection

getConnection方法主要从datasource中获取连接，为了支持事务需要将连接和线程通过TransactionSynchronizationManager绑定，这样确保在spring事务中同一线程获得同一连接。

```
public static Connection getConnection(DataSource dataSource) throws CannotGetJdbcConnectionException {
   try {
      return doGetConnection(dataSource);
   }
   catch (SQLException ex) {
      throw new CannotGetJdbcConnectionException("Failed to obtain JDBC Connection", ex);
   }
   catch (IllegalStateException ex) {
      throw new CannotGetJdbcConnectionException("Failed to obtain JDBC Connection: " + ex.getMessage());
   }
}
```

DataSourceUtils#doGetConnection：

```
	public static Connection doGetConnection(DataSource dataSource) throws SQLException {
		//从TransactionSynchronizationManger中根据datasource获取conHolder
		ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
		//返回conHolder中的连接
		if (conHolder != null && (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {
			conHolder.requested();
			if (!conHolder.hasConnection()) {
				conHolder.setConnection(fetchConnection(dataSource));
			}
			return conHolder.getConnection();
		}
		
		//从datasource中获取连接
		Connection con = fetchConnection(dataSource);
		//如果使用spring事务
		if (TransactionSynchronizationManager.isSynchronizationActive()) {
			try {			
				ConnectionHolder holderToUse = conHolder;
				if (holderToUse == null) {
					holderToUse = new ConnectionHolder(con);
				}
				else {
					holderToUse.setConnection(con);
				}
				//连接的引用数加+
				holderToUse.requested();
				//将连接放入到TransactionSynchronizationManager中，确保在事务中同一线程获取的是同一连接
				TransactionSynchronizationManager.registerSynchronization(
						new ConnectionSynchronization(holderToUse, dataSource));
				holderToUse.setSynchronizedWithTransaction(true);
				if (holderToUse != conHolder) {
					TransactionSynchronizationManager.bindResource(dataSource, holderToUse);
				}
			}
			catch (RuntimeException ex) {
				// Unexpected exception from external delegation call -> close Connection and rethrow.
				releaseConnection(con, dataSource);
				throw ex;
			}
		}

		return con;
	}
```

DataSourceUtils#fetchConnection：

```
	private static Connection fetchConnection(DataSource dataSource) throws SQLException {
		//从datasource中获取connection
		Connection con = dataSource.getConnection();
		if (con == null) {
			throw new IllegalStateException("DataSource returned null from getConnection(): " + dataSource);
		}
		return con;
	}
```

.ResourceHolderSupport#requested：

```
	public void requested() {
		this.referenceCount++;
	}
```



#### JdbcTemplate#applyStatementSettings：

```
	protected void applyStatementSettings(Statement stmt) throws SQLException {
		//获取resultset的next返回的结果个数
		int fetchSize = getFetchSize();
		if (fetchSize != -1) {
			stmt.setFetchSize(fetchSize);
		}
		//设置resultset返回记录的最大行数
		int maxRows = getMaxRows();
		if (maxRows != -1) {
			stmt.setMaxRows(maxRows);
		}
		//设置statement的超时时间
		DataSourceUtils.applyTimeout(stmt, getDataSource(), getQueryTimeout());
	}
```

#### DataSourceUtils#applyTimeout：

```
	public static void applyTimeout(Statement stmt, @Nullable DataSource dataSource, int timeout) throws SQLException {
		Assert.notNull(stmt, "No Statement specified");
		ConnectionHolder holder = null;
		//如果datasource不为空，利用TransactionSynchronizationManager根据datasource获取connectionHolder
		if (dataSource != null) {
			holder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
		}
		//设置statement的超时时间
		if (holder != null && holder.hasTimeout()) {
			stmt.setQueryTimeout(holder.getTimeToLiveInSeconds());
		}
		else if (timeout >= 0) {
			stmt.setQueryTimeout(timeout);
		}
	}
```

#### PreparedStatmentCallback的实现类是lambda函数实现的

```
	protected int update(final PreparedStatementCreator psc, @Nullable final PreparedStatementSetter pss)
			throws DataAccessException {

		return updateCount(execute(psc, ps -> {
			try {
				//doInPreparedStatament函数的逻辑
				if (pss != null) {
					//给preparedstatement设置值
					pss.setValues(ps);
				}
				//preparedstatment执行语句
				int rows = ps.executeUpdate();
				return rows;
			}
			finally {
				if (pss instanceof ParameterDisposer) {
					((ParameterDisposer) pss).cleanupParameters();
				}
			}
		}, true));
	}
}
```

### JdbcTemplate#query原理

JdbcTemplate#query：

```
	public <T> List<T> query(String sql, RowMapper<T> rowMapper) throws DataAccessException {
		//将rowMapper封装到RowMapperResultSetExtractor
		return result(query(sql, new RowMapperResultSetExtractor<>(rowMapper)));
	}
```

JdbcTemplate#query：

```
	public <T> T query(final String sql, final ResultSetExtractor<T> rse) throws DataAccessException {
		//创建内部类QueryStatmentCallback，封装主要的查询逻辑
		class QueryStatementCallback implements StatementCallback<T>, SqlProvider {
			@Override
			@Nullable
			public T doInStatement(Statement stmt) throws SQLException {
				ResultSet rs = null;
				try {
					//执行查询语句
					rs = stmt.executeQuery(sql);
					//利用rowmapper将resultset中的结果封装为pojo
					return rse.extractData(rs);
				}
				finally {
					JdbcUtils.closeResultSet(rs);
				}
			}
			@Override
			public String getSql() {
				return sql;
			}
		}
		//和update中的处理逻辑相同
		return execute(new QueryStatementCallback(), true);
	}
```

#### JdbcTemplate.queryForObject原理

JdbcTemplate.queryForObject：

```
	public <T> T queryForObject(String sql, Class<T> requiredType) throws DataAccessException {
		return queryForObject(sql, getSingleColumnRowMapper(requiredType));
	}
```

JdbcTemplate#getSingleColumnRowMapper：

```
	protected <T> RowMapper<T> getSingleColumnRowMapper(Class<T> requiredType) {
		return new SingleColumnRowMapper<>(requiredType);
	}
```

SingleColumnRowMapper#mapRow：

```
	public T mapRow(ResultSet rs, int rowNum) throws SQLException {
		// Validate column count.
		ResultSetMetaData rsmd = rs.getMetaData();
		int nrOfColumns = rsmd.getColumnCount();
		if (nrOfColumns != 1) {
			throw new IncorrectResultSetColumnCountException(1, nrOfColumns);
		}

		// 将第一列中的值取出，格式转换后返回
		Object result = getColumnValue(rs, 1, this.requiredType);
		if (result != null && this.requiredType != null && !this.requiredType.isInstance(result)) {
			// Extracted value does not match already: try to convert it.
			try {
				return (T) convertValueToRequiredType(result, this.requiredType);
			}
			catch (IllegalArgumentException ex) {
				throw new TypeMismatchDataAccessException(
						"Type mismatch affecting row number " + rowNum + " and column type '" +
						rsmd.getColumnTypeName(1) + "': " + ex.getMessage());
			}
		}
		return (T) result;
	}
```

SingleColumnRowMapper可以将resultset中各个行的第1列的值封装为pojo。