在MyBatis中MappedStatement用于映射一个sql语句的xml节点，如下：
```xml
  <select id="selectByPrimaryKey" parameterType="java.lang.Long" resultMap="BaseResultMap">
    select 
    <include refid="Base_Column_List" />
    from unbind_phone_check
    where id = #{id,jdbcType=BIGINT}
  </select>
```

