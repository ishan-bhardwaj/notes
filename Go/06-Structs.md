# Structs

- Defining a struct -
```
type User struct {
    firstName string
    lastName string
    age int
}
```

> [!NOTE]
> `User` is capitalized so that it can be used in other files.

- Instantiating struct -
```
var appUser User

appUser = User {
    firstName: "John",
    lastName: "Doe",
    age: 30
}
```

> [!TIP]
> We can omit the keys while instantiating the struct if the order of the fields is same as defined in the struct definition.

- Creating null value for the struct - `User{}`
- We can also omit any fields from the struct instatiation and its value will be replaced by the null value of that field's type.

- To access struct fields, use dot(`.`) notation - `appUser.firstName`
- To mutate the values - `appUser.firstName = "Max"`

> [!TIP]
> Go allows accessing elements of a struct from a pointer of struct directly (without pointer dereferencing) -
> ```
> var u *User
> u = &appUser
> u.firstName               // works
> (*u).firstName            // same as above
> ```

- Attaching methods to structs (`u User` is called as "receiver") -
```
func (u User) outputUserDetails() {                 // Declared right after the struct definition
    fmt.Println(u.firstName, u.lastName, u.age)
}

// Calling the method
appUser.outputUserDetails()
```

- To mutate field values inside struct method, pass the pointer of the struct -
```
func(u *User) unsetFirstName() {
    u.firstName = ""
}

appUser.unsetFirstName()        // No need to change anything while calling
```

- Constructors - utility function that instantiates the struct -
```
func newUser(firstName string, lastName string, age int) *User {
    return &User {
        firstName: firstName,
        lastName: lastName,
        age: age
    }
}
```

> [!NOTE]
> TO access struct and its fields & methods from outside the package, we must start them by capital letters.

- Commonly, we define the structs in their own package, say `user`, and name the constructor function as `New`, so we can call it as - `user.New(...)`

## Struct Embedding

- Nesting structs -
```
type Admin struct {
    email string
    password string
    AppUser User 
}
```

- Instantiating -
```
Admin {
    email: "john@gmail.com",
    password: "*****",
    AppUser: User {
        firstName: "John",
        lastName: "Doe",
        age: 30
    }
}
```

> [!TIP]
> To access methods from `User` via `Admin` - `admin.AppUser.OutputUserDetails()`.
> To avoid repetition of `AppUser` (called **anonymous embedding**), we need to only write `User` & not `AppUser User` -
> ```
> type Admin struct {
>    email string
>    password string
>    User
> }
>
> admin.OutputUserDetails()
> ```

## Creating other custom types

- Type aliases -
```
type str string

var name str
```

- Adding methods -
```
func (text str) log() {
    <...>
}

var name str = "Max"
name.log()
```

## Struct Tags

- Metadata added to struct - useful in cases like json marshalling -
```
type User struct {
    firstName string `json:"first_name"`
    lastName string `json:"last_name"`
    age int
}
```
