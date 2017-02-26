---
layout: post
title:  试用nginx stream模块
date:   2014-04-10 23:10:30
tags:
- nginx
---

MyBatis物理分页及Cache总记录数
##原理
没有定义接口，直接通过SqlSessionTemplate执行对应的SQLID，根据mapper.xml中的设置决定是否cache。查询数据用mybatis提供的API即可。查询总记录条数时需要将cache结果。与数据保持一致性这里主要是通过查看cachingexecutor里面的代码实现的。在cache结直接从当前查询数据的MappedStatement获取cache，然后直接扔进去。验证是否更新cache也使用获取数据的MappedStatement来验证。即可达到cache条数与cache数据保持一致性。此处差不多照搬了两个工具方法，一个是创建cachekey，另一个就是设置prepareStatment中的参数的。另外。sqlsessiontemplate是一个代理对象。它提供的操作全是有内部的proxy对象（defaultsqlsession）实现，因为sqlsession是线程不安全的，如果你像我一样一开始就直接通过SqlSessionFactory去opensession你会发现cache是不会生效的。因为每次获取的东东不是单利，而sqlsessiontemplate就是了。。。SqlSessionTemplate本身也实现了SqlSession，她内部的sqlSessionProxy也是一个SqlSession的实现。所以它内部的方法基本都是调用代理对象来实现得。通过sqlsessiontemplate获取java.sql.Connection然后去调用诸如prepareStatement等方法时你会大失所望！

mybatis通过接口执行insert的顺序。通过sqsessiontemplate要比这个简单。不需要接口代理层.
在业务方法中首先调用Mapper接口代理类MapperProxy执行接口中定义的方法MapperMethod，之后在mappermethod中会调用sqlsessiontemplate来完成操作。sqlsessiontemplate将操作交给代理sqlsession（defaultSqlSession），代理sqlsession中会有一个Executor的实现，如果使用了cache，那么这个executor的实现就是cachingexecutor，具体是什么实现根据ExecutorType决定（Executor里面又是代理，名叫SimpleExecutor，晕死人叻），simpleExecutor内部通过handler与数据交互。

##部分代码

```
private static Integer queryTotalCount(String statement, Object parameter, SqlSession session) {
        // cache分页数据
        MappedStatement ms = session.getConfiguration().getMappedStatement( statement);
        BoundSql boundSql = ms.getBoundSql(parameter);
       
        Cache cache = ms.getCache();
       
        String countSql = "select count(1) from (" + ms.getBoundSql(parameter).getSql() + ") tmptb";
        //countBoundSql仅用来创建cachekey
        BoundSql countBoundSql = new BoundSql(session.getConfiguration(), countSql, boundSql.getParameterMappings(), parameter);
        CacheKey key = createCacheKey(ms, parameter, RowBounds.DEFAULT, countBoundSql);
        /**
         * @see org.apache.ibatis.executor.CachingExecutor#query(MappedStatement, Object, RowBounds, ResultHandler, CacheKey, BoundSql)
         * 决定是否从cache中取得数据
         */
        if (cache != null && !ms.isFlushCacheRequired() && ms.isUseCache()) {
            cache.getReadWriteLock().readLock().lock();
            try {
                Integer cachedCount = (Integer) cache.getObject(key);
                if (cachedCount != null){
                    if(log.isDebugEnabled())
                        log.debug(String.format("从cache中获取总记录数[%s]",cachedCount));
                    return cachedCount;
                }
            } finally {
                cache.getReadWriteLock().readLock().unlock();
            }
        }
        try {
            PreparedStatement pstmt = SqlSessionUtils.getSqlSession(getBean(SqlSessionFactory.class)).getConnection().prepareStatement( countSql);
            setParameter(boundSql,session,parameter,ms,pstmt);
            ResultSet rs = pstmt.executeQuery();
            int totalCount = 0;
            if (rs.next()) {
                totalCount = rs.getInt(1);
                if (ms.getCache() != null) cache.putObject(key, totalCount);
            }
            return totalCount;
        } catch (Exception e) {
            throw new RuntimeException("查询记录数出错", e);
        }
    }
    /**
     * @see org.apache.ibatis.executor.BaseExecutor#createCacheKey(MappedStatement, Object, RowBounds, BoundSql)
     * @param ms
     * @param parameterObject
     * @param rowBounds
     * @param boundSql
     * @return
     * @author tianjie
     * @date Sep 22, 2013
     * @version v1.0
     */
    private static CacheKey createCacheKey(MappedStatement ms, Object parameterObject,
            RowBounds rowBounds, BoundSql boundSql) {
        CacheKey cacheKey = new CacheKey();
        cacheKey.update(ms.getId());
        cacheKey.update(rowBounds.getOffset());
        cacheKey.update(rowBounds.getLimit());
        cacheKey.update(boundSql.getSql());
        List<ParameterMapping> parameterMappings = boundSql
                .getParameterMappings();
        if (parameterMappings.size() > 0 && parameterObject != null) {
            TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration()
                    .getTypeHandlerRegistry();
            if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                cacheKey.update(parameterObject);
            } else {
                MetaObject metaObject = ms.getConfiguration().newMetaObject(
                        parameterObject);
                for (ParameterMapping parameterMapping : parameterMappings) {
                    String propertyName = parameterMapping.getProperty();
                    if (metaObject.hasGetter(propertyName)) {
                        cacheKey.update(metaObject.getValue(propertyName));
                    } else if (boundSql.hasAdditionalParameter(propertyName)) {
                        cacheKey.update(boundSql
                                .getAdditionalParameter(propertyName));
                    }
                }
            }
        }
        return cacheKey;
    }

    /**
     * 照搬
     * @see org.apache.ibatis.scripting.defaults.DefaultParameterHandler#setParameters(PreparedStatement)
     */
    private static void setParameter(BoundSql boundSql,SqlSession session,Object parameter,MappedStatement ms,PreparedStatement pstmt ) throws Exception{
        List<ParameterMapping> parameterMappings = boundSql .getParameterMappings();
        if (parameterMappings != null) {
            MetaObject metaObject = session.getConfiguration()
                    .newMetaObject(parameter);
            for (int i = 0; i < parameterMappings.size(); i++) {
                ParameterMapping parameterMapping = parameterMappings
                        .get(i);
                if (parameterMapping.getMode() != ParameterMode.OUT) {
                    Object value;
                    String propertyName = parameterMapping.getProperty();
                    if (boundSql.hasAdditionalParameter(propertyName)) {
                        value = boundSql
                                .getAdditionalParameter(propertyName);
                    } else if (parameter == null) {
                        value = null;
                    } else if (ms.getConfiguration()
                            .getTypeHandlerRegistry()
                            .hasTypeHandler(parameter.getClass())) {
                        value = parameter;
                    } else {
                        value = metaObject == null ? null : metaObject
                                .getValue(propertyName);
                    }
                    @SuppressWarnings("rawtypes")
                    TypeHandler typeHandler = parameterMapping
                            .getTypeHandler();
                    JdbcType jdbcType = parameterMapping.getJdbcType();
                    if (value == null && jdbcType == null)
                        jdbcType = ms.getConfiguration()
                                .getJdbcTypeForNull();
                    typeHandler.setParameter(pstmt, i + 1, value, jdbcType);
                }
            }
        }
    }
```

另外：在Spring中配置一个SqlSessionTemplate，在系统卸载SqlSessionTemplate时有异常产生，不用管，sqlsessiontemplate不支持这些操作。原因(这里)[http://lizhizhang.iteye.com/blog/1917807]。
