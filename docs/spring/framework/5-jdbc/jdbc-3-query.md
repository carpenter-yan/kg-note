##query功能实现

###示例
List<User> list = jdbcTemplate.query("select * from user where age=?", 
    new Object[]{20}, new int[]{java.sql.Types.INTEGER}, new UserRowMapper());

```
public <T> List<T> query(String sql, Object[] args, int[] argTypes, RowMapper<T> rowMapper) throws DataAccessException {
    return query(sql, args, argTypes, new RowMapperResultSetExtractor<T>(rowMapper));
}

public <T> T query(String sql, Object[] args, int[] argTypes, ResultSetExtractor<T> rse) throws DataAccessException {
    return query(sql, newArgTypePreparedStatementSetter(args, argTypes), rse);
}

public <T> T query(String sql, PreparedStatementSetter pss, ResultSetExtractor<T> rse) throws DataAccessException {
    return query(new SimplePreparedStatementCreator(sql), pss, rse);
}

public <T> T query(
        PreparedStatementCreator psc, final PreparedStatementSetter pss, final ResultSetExtractor<T> rse)
        throws DataAccessException {

    Assert.notNull(rse, "ResultSetExtractor must not be null");
    logger.debug("Executing prepared SQL query");

    //1. 执行SQL查询， 参考<jdbc-2-excute.md>
    return execute(psc, new PreparedStatementCallback<T>() {
        @Override
        public T doInPreparedStatement(PreparedStatement ps) throws SQLException {
            ResultSet rs = null;
            try {
                if (pss != null) {
                    //2. 设置参数，参考<jdbc-1-update.md>
                    pss.setValues(ps);
                }
                rs = ps.executeQuery();
                ResultSet rsToUse = rs;
                if (nativeJdbcExtractor != null) {
                    rsToUse = nativeJdbcExtractor.getNativeResultSet(rs);
                }
                return rse.extractData(rsToUse);
            }
            finally {
                JdbcUtils.closeResultSet(rs);
                if (pss instanceof ParameterDisposer) {
                    ((ParameterDisposer) pss).cleanupParameters();
                }
            }
        }
    });
}
```
其它，如无参数查询时使用的statement，queryForObject等，实现方式大同小异，此处不再深入。
