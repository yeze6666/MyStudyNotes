### MyBatis plus 和Mybatis plus join的关于多表查询使用

> 一对一
>
> 一对多
>
> 多对多

#### 1. MyBatis plus自己提供了BaseMapper这个实现类，可以方便快速的实现CURD，十分方便！

*但是要注意的是在使用MyBatis Plus的去实现一对一，多对多等的时候，要注意的是，要加入@TableFile(exist = false) 在private User user上！！！(告诉Mybatis Plus这个在数据库上没有该字段！这个只是方便调用第三方表)*，对了注意一下外键不要加@TableFile(exist =false)因为外键本质上也是数据库表上的一个字段！



> 下面是代码学习

```java
// id_card表
@Data
@TableName("id_card") // 绑定数据库表名
public class IdCard {
    // 主键id，自增
    @TableId(type = IdType.AUTO)
    private Integer id;
    
    // 身份证号（对应表中card_no字段，MyBatis-Plus默认下划线转驼峰，无需额外注解）
    private String cardNo;
    
    // 地址（对应表中address字段）
    private String address;
}
```



```java
// User表
@Data
@TableName("user") // 绑定数据库表名
public class User {
    // 主键id，自增
    @TableId(type = IdType.AUTO)
    private Integer id;
    
    // 姓名（对应表中name字段）
    private String name;
    
    // 外键字段：关联身份证表的id（对应表中id_card_id字段）
    // 用于数据库层面存储关联关系，支持MyBatis-Plus自动CRUD
    private Integer idCardId;
    
    // 关联对象：一对一关联的身份证对象
    // 数据库表中无此字段，必须加@TableField(exist = false)避免MyBatis-Plus自动CRUD出错
    @TableField(exist = false)
    private IdCard idCard;
}
```



```java
public interface UserMapper extends BaseMapper<User> {

    /**
     * 自定义方法：查询用户并关联查询身份证信息
     * @param id 用户id
     * @return 包含身份证信息的User对象
     */
    @Select("SELECT * FROM user WHERE id = #{id}") // 查询用户的SQL
    @Results({
        // 映射用户自身字段
        @Result(column = "id", property = "id"),
        @Result(column = "name", property = "name"),
        @Result(column = "id_card_id", property = "idCardId"), // 映射外键字段
        
        // 映射关联的身份证对象（一对一）
        @Result(
            column = "id_card_id", // 用用户表的外键id_card_id作为参数
            property = "idCard",   // 映射到User类的idCard属性
            one = @One(
                select = "com.example.mapper.IdCardMapper.selectById", // 调用身份证的查询方法
                fetchType = FetchType.EAGER // 立即加载（一对一推荐）
            )
        )
    })
    User selectUserWithIdCard(Integer id);
}
```

###### 		到时候就直接调用userMapper.getIdCard();这样可以获取IdCard对应用户的全部信息（身份证号，地址），然后可以通过userMapper.getIdCard().getAddress()或userMapper.getIdCard().getIdCardNumber()就可以获取IdCard中想要的信息了！！！



#### 2.MyBatis Plus Join 

##### but 我是十分懒的，不想使用@Results和@Result，还有@one和@Many这样配合，我不想动脑子！

*所有我想到了MyBatis Plus Join ！*

为什么使用它？？？

因为它不用写那那些烦人的注解，它要改动的地方也很少！十分适合我这种懒人，哈哈哈哈哈



只需要在

`public interface UserMapper extends BaseMapper<User>`

这里，将BaseMapper改为MPJBaseMapper ！！！就🆗了！！！

同时不用写那些@Results和@Result等等的注解

> 话不多说，代码展示

```java
// 用户实体
@TableName("user")
public class User {
    @TableId(type = IdType.AUTO)
    private Integer id;
    private String name;
    private Integer idCardId; // 外键

    @TableField(exist = false) // 非表字段
    private IdCard idCard; // 关联对象

    // get/set
}

// 身份证实体
@TableName("id_card")
public class IdCard {
    @TableId(type = IdType.AUTO)
    private Integer id;
    private String cardNo;
    private String address;

    // get/set
}
```



````java
@Service
public class UserService {
    @Autowired
    private UserMapper userMapper;

    // 查询用户及其所有订单（一对多）
    public User getUserWithOrders(Integer userId) {
        MPJLambdaWrapper<User> wrapper = new MPJLambdaWrapper<User>()
                .selectAll(User.class)
                // 关联订单表（一对多）
                .leftJoin(Order.class, Order::getUserId, User::getId)
                // 订单表查询字段（映射到User.orders）
                .select(Order::getId, Order::getOrderNo, Order::getPrice)
                // 按用户id分组（确保一个用户只返回一条记录，订单封装为集合）
                .groupBy(User::getId)
                .eq(User::getId, userId);

        // 执行查询
        return userMapper.selectOne(wrapper);
    }
}
````



| 方法名                   | 作用                     | 示例                                                   |
| ------------------------ | ------------------------ | ------------------------------------------------------ |
| `selectAll(类)`          | 查询主表所有字段         | `selectAll(User.class)`                                |
| `leftJoin(类, 关联条件)` | 左连接关联表             | `leftJoin(Order.class, Order::getUserId, User::getId)` |
| `select(字段...)`        | 选择关联表需要查询的字段 | `select(Order::getId, Order::getOrderNo)`              |
| `eq(字段, 值)`           | 添加等于条件             | `eq(User::getId, 1)`                                   |
| `groupBy(字段)`          | 按字段分组（一对多必备） | `groupBy(User::getId)`                                 |

但是要注意的是它的底层使用的是JOIN，小表还可以，但是大表就一定不用使用！