## 一种简单灵活的代码结构设计

数据模型不依赖项目中其他模块，作为基础模块。

比如，我们有个用户实体：

```go
type User struct {
	UserName string
	ID string
}
```


在同一模块中，设计针对user的操作的接口，不针对具体实现：
```go
type UserServicer interface {
	User(id int) (*User, error)
	CreateUser(u *User) error
	DeleteUser(id int) error
}
```

userService具体的实现就在另外一个地方。

比如存储在postgres中的实现UserService
```go
type UserService struct {
	DB *sql.DB
}

Func (us *UserService) User(id int)(*User, error){
	***
	return user
}
...
```

这样我们还可以实现其他存储的实现，比如缓存。

```go
type UserCache struct {
	cache map[int]*User
	service UserService
}

Func (uc *UserCache)User(id int)(*User, error){
	if u := u.cache[id];u!= nil {
		return u, nil
	}else{
		user, err := uc.service.User(id)
        if err != nil {
            return nil, err
        }
        u.cache[id] = user       
	}
}
...
```

这样我们就很方便的给user查询加了缓存层。

假如有些数据还需要变更到第三方服务中，我们还可以把这些三方服务抽象成interface，这样我们还可以再增加中间层。

使用抽象的interface后，我们还可以不使用各种mock库，自己来很简单的mock。
```go
type UserServiceMock struct {
	UserFn User(id int) (*User, error)
}

func (usm *UserServiceMock)User(id int)(*User, error){return usm.UserFn(id)}
...

```

单元测试中:
```go
userMock := new(UserServiceMock)
userMock.UserFn = func(id int) (*User, error){
	return ***, nil
}
```
这样就简单明了的在我们的mock中构造自己想要的数据了。