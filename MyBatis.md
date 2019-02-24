#### **映射文件参数处理** 

**单个参数**：直接写 #{参数名}

**多个参数：**Mybatis将多个入参封装成一个map，key为param1,param2...paramN，value为对应的值

所以取值时应写为 #{key}

```xml
<select id="getEmployee" resultType="pojo.employee">
    select id, lastname, email from table_employee 
    where id = #{param1} and lastname like #{param2}
</select>
```

或者在写 Mapper接口方法时指定封装成 map时的key

```java
public List<Employee> getEmployee(@Param("id")Integer id, @Param("lastname")String lastname)
```

```xml
<select id="getEmployee" resultType="pojo.employee">
    select id, lastname, email from table_employee 
    where id = #{id} and lastname like #{lastname}
</select>
```

**pojo & map & DTO :**   #{属性名}    #{key}

非业务模型的数据，如果不常用就传入一个map，如果经常使用推荐编写一个DTO



#### **resultMap属性**

在java bean属性与字段列名不一致时使用，与resultType属性不能同时使用

**自定义映射规则** 

```xml
<resultMap type="bean.Employee" id="MyEmp">
    <!--指定主键列的封装规则，column是字段，property是bean属性-->
    <id column="id" property="id" />
    <!--定义普通列封装规则-->
    <result column="is_left" property="left" />
    <!--其他名称一致的列可以不写-->
</resultMap>

<select id="getEmployee" resultMap="MyEmp">
</select>
```

**关联查询**（嵌套对象，级联属性）

```xml
<!--同时查询出员工及其部门信息-->
<!--方法1、级联属性封装结果，dept是Employee类的一个内部属性-->
<resultMap type="bean.Employee" id="MyEmpAndDept">
    <id column="id" property="id" />
    <result column="last_name" property="lastName" />
    <result column="dept_id" property="dept.id" />                
    <result column="dept_name" property="dept.deptName" />
</resultMap>

<select id="getEmpAndDept" resultMap="MyEmpAndDept">
    select e.id id, e.last_name last_name, e.dept_id dept_id, d.dept_name dept_name
    from table_employee e, table_department d
    where e.dept_id = d.id and e.id = #{id} 
</select>

<!--方法2、association指定联合的javaBean对象，嵌套进去-->
<resultMap type="bean.Employee" id="MyEmpAndDept">
    <id column="id" property="id" />
    <result column="last_name" property="lastName" />
    <association property="dept" javaType="bean.Department">
        <id column="dept_id" property="id" />
        <result column="dept_name" property="deptName" />
    </association>
</resultMap>
```

**分步查询**（association）

```xml
<!--先查出员工信息，根据员工信息中的dept_id查出部门信息，封装成对象赋给Employee对象中的dept属性-->
<!--association指定关联对象的封装规则，
	select：该属性是由哪个方法查询得到；column：传入该方法的是哪一列的值-->
<resultMap type="bean.Employee" id="MyEmpAndDept">
    <id coloumn="id" property="id" />
    <result column="last_name" property="lastName" />
    <association property="dept"
                 select="dao.mapper.DeptMapper.getDeptById"
                 column="dept_id">
    </association>
</resultMap>
    
<select id="getEmpByStep" resultMap="MyEmpAndDept">
    select id, last_name, dept_id from table_employee where id = #{id}
</select>
```

**分步查询延迟加载**

```properties
#部门信息在需要的时候才去加载，不需要时仅做员工信息的查询
#在Mybatis的配置中设置两个属性(application.properties)
mybatis.configuration.lazy-loading-enabled=true
mybatis.configuration.aggressive-lazy-loading=false      

#根据以上配置，只有在需要使用级联属性时（例如employee.getDept().getDeptName）才会执行对部门的查询
```

**关联集合封装**（collection）

```xml
<!--查询一个部门及其所有员工，List<Employee>作为Department的属性-->
<!--collection定义关联集合类型的封装规则，ofType表示集合元素的类型-->
<resultMap type="bean.Department" id="MyDept">
    <id column="dept_id" property="id"/>
    <result column="dept_name" property="deptName"/>
    <collection property="emps" ofType="bean.Employee">
        <id column="emp_id" property="id" />
    	<result column="last_name" property="lastName" />
    </collection>
</resultMap>

<select id="getDeptAndAllEmps" resultMap="MyDept">
    select d.id dept_id, d.dept_name dept_name, e.id emp_id, e.last_name last_name
    from table_department d 
    left join table_employee e
    on d.id = e.dept_id where d.id = #{deptId}
</select>

<!--同样可以分步查询，按需加载，这样sql语句只需要写查询部门-->
<resultMap type="bean.Department" id="MyDept">
    <id column="id" property="id"/>
    <result column="dept_name" property="deptName"/>
    <collection property="emps" 
                select="dao.mapper.EmployeeMapper.getEmpsByDeptId"
                column="id">
    </collection> 
</resultMap>    
<select id="getDeptByStep" resultMap="MyDept">
    select id, dept_name from table_department where id = #{deptId}
</select>
```

**分步查询传递多列值**

```xml
	<collection property="emps" 
                select="dao.mapper.EmployeeMapper.getEmpsByDeptId"
                column="{deptId=id, ...}">                <!--"{key1=column1, key2=column2}"-->
    </collection> 
```

**注解版的resultMap** 

@Results相当于resultMap标签；@Result相当于result标签；@One相当于association标签；@Many相当于collection标签，@Many前面的column相当于分步查询的连接列

 ```java
@Results({
    @Result(column="id", property="id"),
    @Result(column="dept_name", property="deptName"),
    @Result(property="emps", column="id", many=@Many(select="dao.mapper.EmployeeMapper.getEmpsByDeptId"))
})
@Select("select id, dept_name from table_department where id = #{deptId}")
public Department getDeptAndAllEmps(Integer deptId);
 ```



#### **动态sql**

（if、trim、choose、foreach、bind）

```xml
<mapper namespace="dao.EmployeeMapper">
    <!--1、入参的POJO携带了哪个字段，查询条件就带哪个字段（id是接口中的方法名）-->
    <select id="getEmpsByConditionIf" resultType="pojo.employee">
        select id, lastname, email, gender from table_employee
        where 1=1                                              <!--避免sql拼接时出现冗余的and-->
        <if test="id!=null">
            and id = #{id}
        </if>
        <if test="lastname!=null and lastname!=&quot;&quot;">     <!--特殊符号用转义字符-->
            and lastname like #{lastname}
        </if>
        <if test="gender==0 or gender==1">                <!--OGNL会自动将字符串转为数字-->
            and gender = #{gender}
        </if>
    </select>
    
    <!--2、也可以使用trim标签解决多余的and问题，如果trim里面的条件都不满足，则查询全部
        prefix是指加上前缀，prefixOverrides是指去掉前面多余的前缀-->
    <select id="getEmpsByConditionTrim" resultType="pojo.employee">
        <trim prefix="where" prefixOverrides="and">
            <if test="id!=null"> id = #{id} </if>
            <if test="lastname!=null"> and lastname like #{lastname} </if>
        </trim>
    </select>
 
    <!--3、set与if、trim配合实现更新-->
    <update id="updateEmpsByConditionTrim">
        update table_employee
        <trim prefix="set" suffixoverride="," suffix=" where id = #{id} ">
            <if test="lastname != null and lastname.length()>0"> lastname=#{lastname}, </if>
            <if test="gender ==0 or gender==1"> gender=#{gender} </if>
        </trim>
	</update>
    <!--直接使用<set>标签也可以忽略冗余逗号-->
    <update id="updateEmpsByConditionTrim">
        update table_employee
        <set>                                                
            <if test="lastname != null and lastname.length()>0"> lastname=#{lastname}, </if>
            <if test="gender ==0 or gender==1"> gender=#{gender} </if>
        </set>
        where id = #{id}
	</update>
    
    <!--4、choose(when, otherwise)-->
    <select id="getEmployeeByConditionChoose" resultType="pojo.employee">
        select id, lastname from table_employee
        <where>
            <when test="id > 0"> id = #{id} </when>
            <when test="lastname.length() > 0"> lastname like #{lastname} </when>
            <otherwise> 1=1 </otherwise>
        </where>
    </select>
    
    <!--5、foreach实现集合遍历的查询-->
    <select id="getEmployeeByConditionForeach" resultType="pojo.employee">
        select id, lastname from table_employee where id in 
        <foreach collection="idlist" item="item_id" separator=","
                 open="(" close=")">
            #{item_id}
        </foreach>
    </select>
    
    <!--6、foreach实现批量保存-->
    <!--MySQL版本-->
    <insert id="saveEmployeeInBatchForMysql">
        insert into table_employee (id, lastname, email, gender) values
        <foreach collection="emps" item="emp" separator=",">
            (#{emp.id}, #{emp.lastname}, #{emp.email}, #{emp.gender})
        </foreach>
    </insert>    
    <!--Oracle版本
		Oracle支持的批量插入方式：1、多个insert语句放在begin - end里面；2、利用中间表-->
    <insert id="saveEmployeeInBatchForOracle">
        <foreach collection="emps" item="emp" open="begin" close="end">
        	insert into table_employee (id, lastname, email, gender) values
            (id_seq.nextval, #{lastname}, #{emp.gender});
        </foreach>
    </insert>
    
    <!--7、用 bind 将OGNL的值绑定到一个变量中，方便后来引用-->
    <select id="getEmployeeUsingBind" resultType="pojo.employee">
        <bind name="_lastname" value=" '%' + lastname + '%' " />
        select id, lastname, email from table_employee where id = #{_lastname}
    </select>
    
    <!--8、抽取可重用的sql片段-->
    <sql id = "selectHead">
        select id, lastname, email, gender from table_employee
    </sql>
    <select id="getEmployeeUsingRef" resultType="pojo.employee">
        <include refid="selectHead"></include> where id = #{id}
    </select>
</mapper>
```

