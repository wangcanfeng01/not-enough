# 防忘记
mybatis执行数据操作都是通过Executor执行的,由sessionFactory创建对应的类型的Executor，默认的ExecutorType是SIMPLE，所以如果需要创建BatchExecutor的话，就需要自己设置本次session，传入BATCH操作类型。
``` java
// 引入template
@Autowired
private SqlSessionTemplatesqlSessionTemplate;
public void saveAndUpdate(List list)throws Exception {
// 当前会话开启批量模式
SqlSession sqlSession =sqlSessionTemplate.getSqlSessionFactory().openSession(ExecutorType.BATCH, false);
// 从会话中获取到mapper
    CustomerInsightMapper mapper = sqlSession.getMapper(CustomerInsightMapper.class);
    try {
for (CustomerInsightInfo info : list) {
mapper.saveAndUpdate(info);
        }
//提交
sqlSession.commit();
        sqlSession.clearCache();
    }catch (Exception e) {
sqlSession.rollback();
        throw new Exception(e);
    }finally {
sqlSession.close();
    }
}
```
原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)