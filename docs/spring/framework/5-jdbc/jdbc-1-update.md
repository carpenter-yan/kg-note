##spring jdbc

### save/update
用法示例
```
public class UserServiceImpl implements UserService {
    private JdbcTemplate jdbcTemplate;
    //设置数据源
    public void setDataSource(DataSource dataSource) {
        this.jdbcTemplate = new JdbcTemplate(dataSource);
    }
    public void save(User user) {
    jdbcTemplate.update("insert into user(name , age , sex)values(? , ?, ?)",
    new Object [] {user.getName(), user.getAge(), user.get Sex()} , 
    new int[] {Java.sql.Types.VARCHAR, java.sql.Types.INTEGER, java.sql.Types.VARCHAR}) ;
    
    @SuppressWarnings ("unchecked")
    public List<User> get Users () {
    List<User> list = jdbcTemplate.query ("select * from user", new UserRowMapper());
        return list ;
    }
}
```

JdbcTemplate.update
```
// 将参数封装为ArgumentTypePreparedStatementSetter
public int update(String sql, Object[] args, int[] argTypes) throws DataAccessException {
    return update(sql, newArgTypePreparedStatementSetter(args, argTypes));
}

// 将sql语句封装为SimplePreparedStatementCreator
public int update(String sql, PreparedStatementSetter pss) throws DataAccessException {
    return update(new SimplePreparedStatementCreator(sql), pss);
}

protected int update(final PreparedStatementCreator psc, final PreparedStatementSetter pss)
        throws DataAccessException {

    logger.debug("Executing prepared SQL update");
    return execute(psc, new PreparedStatementCallback<Integer>() {
        @Override
        public Integer doInPreparedStatement(PreparedStatement ps) throws SQLException {
            try {
                if (pss != null) {
                    //1. 设置参数
                    pss.setValues(ps);
                }
                int rows = ps.executeUpdate();
                if (logger.isDebugEnabled()) {
                    logger.debug("SQL update affected " + rows + " rows");
                }
                return rows;
            }
            finally {
                if (pss instanceof ParameterDisposer) {
                    ((ParameterDisposer) pss).cleanupParameters();
                }
            }
        }
    });
}

ArgumentTypePreparedStatementSetter.setValues
public void setValues(PreparedStatement ps) throws SQLException {
    int parameterPosition = 1;
    if (this.args != null) {
        for (int i = 0; i < this.args.length; i++) {
            Object arg = this.args[i];
            if (arg instanceof Collection && this.argTypes[i] != Types.ARRAY) {
                Collection<?> entries = (Collection<?>) arg;
                for (Object entry : entries) {
                    if (entry instanceof Object[]) {
                        Object[] valueArray = ((Object[]) entry);
                        for (Object argValue : valueArray) {
                            doSetValue(ps, parameterPosition, this.argTypes[i], argValue);
                            parameterPosition++;
                        }
                    }
                    else {
                        doSetValue(ps, parameterPosition, this.argTypes[i], entry);
                        parameterPosition++;
                    }
                }
            }
            else {
                //1. 设置参数
                doSetValue(ps, parameterPosition, this.argTypes[i], arg);
                parameterPosition++;
            }
        }
    }
}

StatementCreatorUtils.setParameterValueInternal
private static void setParameterValueInternal(PreparedStatement ps, int paramIndex, int sqlType,
        String typeName, Integer scale, Object inValue) throws SQLException {

    String typeNameToUse = typeName;
    int sqlTypeToUse = sqlType;
    Object inValueToUse = inValue;

    // override type info?
    if (inValue instanceof SqlParameterValue) {
        SqlParameterValue parameterValue = (SqlParameterValue) inValue;
        if (logger.isDebugEnabled()) {
            logger.debug("Overriding type info with runtime info from SqlParameterValue: column index " + paramIndex +
                    ", SQL type " + parameterValue.getSqlType() + ", type name " + parameterValue.getTypeName());
        }
        if (parameterValue.getSqlType() != SqlTypeValue.TYPE_UNKNOWN) {
            sqlTypeToUse = parameterValue.getSqlType();
        }
        if (parameterValue.getTypeName() != null) {
            typeNameToUse = parameterValue.getTypeName();
        }
        inValueToUse = parameterValue.getValue();
    }

    if (logger.isTraceEnabled()) {
        logger.trace("Setting SQL statement parameter value: column index " + paramIndex +
                ", parameter value [" + inValueToUse +
                "], value class [" + (inValueToUse != null ? inValueToUse.getClass().getName() : "null") +
                "], SQL type " + (sqlTypeToUse == SqlTypeValue.TYPE_UNKNOWN ? "unknown" : Integer.toString(sqlTypeToUse)));
    }

    if (inValueToUse == null) {
        setNull(ps, paramIndex, sqlTypeToUse, typeNameToUse);
    }
    else {
        //1. 设置参数
        setValue(ps, paramIndex, sqlTypeToUse, typeNameToUse, scale, inValueToUse);
    }
}

// 根据不同的参数类型进行设值
private static void setValue(PreparedStatement ps, int paramIndex, int sqlType, String typeName,
        Integer scale, Object inValue) throws SQLException {

    if (inValue instanceof SqlTypeValue) {
        ((SqlTypeValue) inValue).setTypeValue(ps, paramIndex, sqlType, typeName);
    }
    else if (inValue instanceof SqlValue) {
        ((SqlValue) inValue).setValue(ps, paramIndex);
    }
    else if (sqlType == Types.VARCHAR || sqlType == Types.NVARCHAR ||
            sqlType == Types.LONGVARCHAR || sqlType == Types.LONGNVARCHAR) {
        ps.setString(paramIndex, inValue.toString());
    }
    else if ((sqlType == Types.CLOB || sqlType == Types.NCLOB) && isStringValue(inValue.getClass())) {
        String strVal = inValue.toString();
        if (strVal.length() > 4000) {
            // Necessary for older Oracle drivers, in particular when running against an Oracle 10 database.
            // Should also work fine against other drivers/databases since it uses standard JDBC 4.0 API.
            try {
                if (sqlType == Types.NCLOB) {
                    ps.setNClob(paramIndex, new StringReader(strVal), strVal.length());
                }
                else {
                    ps.setClob(paramIndex, new StringReader(strVal), strVal.length());
                }
                return;
            }
            catch (AbstractMethodError err) {
                logger.debug("JDBC driver does not implement JDBC 4.0 'setClob(int, Reader, long)' method", err);
            }
            catch (SQLFeatureNotSupportedException ex) {
                logger.debug("JDBC driver does not support JDBC 4.0 'setClob(int, Reader, long)' method", ex);
            }
        }
        // Fallback: regular setString binding
        ps.setString(paramIndex, strVal);
    }
    else if (sqlType == Types.DECIMAL || sqlType == Types.NUMERIC) {
        if (inValue instanceof BigDecimal) {
            ps.setBigDecimal(paramIndex, (BigDecimal) inValue);
        }
        else if (scale != null) {
            ps.setObject(paramIndex, inValue, sqlType, scale);
        }
        else {
            ps.setObject(paramIndex, inValue, sqlType);
        }
    }
    else if (sqlType == Types.BOOLEAN) {
        if (inValue instanceof Boolean) {
            ps.setBoolean(paramIndex, (Boolean) inValue);
        }
        else {
            ps.setObject(paramIndex, inValue, Types.BOOLEAN);
        }
    }
    else if (sqlType == Types.DATE) {
        if (inValue instanceof java.util.Date) {
            if (inValue instanceof java.sql.Date) {
                ps.setDate(paramIndex, (java.sql.Date) inValue);
            }
            else {
                ps.setDate(paramIndex, new java.sql.Date(((java.util.Date) inValue).getTime()));
            }
        }
        else if (inValue instanceof Calendar) {
            Calendar cal = (Calendar) inValue;
            ps.setDate(paramIndex, new java.sql.Date(cal.getTime().getTime()), cal);
        }
        else {
            ps.setObject(paramIndex, inValue, Types.DATE);
        }
    }
    else if (sqlType == Types.TIME) {
        if (inValue instanceof java.util.Date) {
            if (inValue instanceof java.sql.Time) {
                ps.setTime(paramIndex, (java.sql.Time) inValue);
            }
            else {
                ps.setTime(paramIndex, new java.sql.Time(((java.util.Date) inValue).getTime()));
            }
        }
        else if (inValue instanceof Calendar) {
            Calendar cal = (Calendar) inValue;
            ps.setTime(paramIndex, new java.sql.Time(cal.getTime().getTime()), cal);
        }
        else {
            ps.setObject(paramIndex, inValue, Types.TIME);
        }
    }
    else if (sqlType == Types.TIMESTAMP) {
        if (inValue instanceof java.util.Date) {
            if (inValue instanceof java.sql.Timestamp) {
                ps.setTimestamp(paramIndex, (java.sql.Timestamp) inValue);
            }
            else {
                ps.setTimestamp(paramIndex, new java.sql.Timestamp(((java.util.Date) inValue).getTime()));
            }
        }
        else if (inValue instanceof Calendar) {
            Calendar cal = (Calendar) inValue;
            ps.setTimestamp(paramIndex, new java.sql.Timestamp(cal.getTime().getTime()), cal);
        }
        else {
            ps.setObject(paramIndex, inValue, Types.TIMESTAMP);
        }
    }
    else if (sqlType == SqlTypeValue.TYPE_UNKNOWN || (sqlType == Types.OTHER &&
            "Oracle".equals(ps.getConnection().getMetaData().getDatabaseProductName()))) {
        if (isStringValue(inValue.getClass())) {
            ps.setString(paramIndex, inValue.toString());
        }
        else if (isDateValue(inValue.getClass())) {
            ps.setTimestamp(paramIndex, new java.sql.Timestamp(((java.util.Date) inValue).getTime()));
        }
        else if (inValue instanceof Calendar) {
            Calendar cal = (Calendar) inValue;
            ps.setTimestamp(paramIndex, new java.sql.Timestamp(cal.getTime().getTime()), cal);
        }
        else {
            // Fall back to generic setObject call without SQL type specified.
            ps.setObject(paramIndex, inValue);
        }
    }
    else {
        // Fall back to generic setObject call with SQL type specified.
        ps.setObject(paramIndex, inValue, sqlType);
    }
}
```
