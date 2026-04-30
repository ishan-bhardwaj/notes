# Hibernate

- Framework for persisting Java objects in a database.
- Benefits -
  - Hibernate handles all of the low-level SQL.
  - Minimizes the amount of JDBC code you have to develop.
  - Provides the Object-to-Relational Mapping (ORM).

## JPA 

- Jakarta Persistence API - standard for ORM.
- Only a specification -
  - defines a set of interfaces.
  - requires an implementation to be usable.

- In Spring Boot, Hibernate is default implementation of JPA.
- Hibernate / JPA uses internally JDBC for all database communications.

- `EntityManager` (from JPA) is the main component for creating queries etc.

- To setup, add dependencies -
  - MySQL Driver - `mysql-connector-j`
  - Sprint Data JPA - `spring-boot-starter-data-jpa`

> [!NOTE]
> Spring Boot automatically configures your data source based on entries in Maven pom file. Also, Spring boot reads the database connection info from `application.properties` file.

- `application.properties` -
```
spring.datasource.url=jdbc://mysql://localhost:3306/students
spring.datasource.username=root
spring.datasource.password=root@123
```

> [!NOTE]
> There is no need to give JDBC driver class name. Spring Boot will automatically detect it based on the URL.

> [!TIP]
> To turn off Spring Boot banner that comes on top of the logs - add `spring.main.banner-mode=off` in `application.properties`.

> [!TIP]
> To set Spring Boot's logger - `logging.level.root=warn` in `application.properties`.
> To log Hibernate SQL statements - `logging.level.org.hibernate.SQL=debug`
> To log values for SQL statements - `logging.level.org.hibernate.orm.jdbc.bind=trace`

## Entity class

- Java class that is mapped to a database table.
- The entity class -
  - Must be annotated with `@Entity`.
  - Must have a public or protected no-argument constructor. It can have other constructors.

```
@Entity
@Table(name="student")
public class Student {

  @Id
  @Column(name="id")
  private int id;

  @Column(name="first_name")
  private String firstName;

}
```

> [!NOTE] 
> `@Table` & `@Column` are optional and if they are not specified, the table / column name will have the same name as the Java Class / field.

## Id generation methods

| Name                    | Description                                                                        |
|-------------------------|------------------------------------------------------------------------------------|
| GenerationType.AUTO     | Picks an appropriate strategy for the particular database                          |
| GenerationType.IDENTITY | Assign primary keys using database identity column                                 |
| GenerationType.SEQUENCE | Assign primary keys using a database sequence                                      |
| GenerationType.TABLE    | Assign primary keys using an underlying database table to ensure unqiueness        |
| GenerationType.UUID     | Assign primary keys using a globally unique identifier (UUID) to ensure uniqueness |

- Specifying id generation methods -
```
@Id
@GeneratedValue(strategy=GenerationType.IDENTITY)
@Column(name="id")
private int id;
```

- Defining custom generation strategy -
  - Implement - `org.hibernate.id.IdentifierGenerator`
  - Override the method - `public Serializable generate(...)`

## Data Access Object (DAO)

- Responsible for interfacing with the database.
- DAO needs a JPA Entity Manager which is the main component for saving/retrieving entities.
- JPA further needs a Data Source which defines database connection info.
- JPA Entity Manager and Data Source are automatically created by Sprint Boot - based on `application.properties` (JDBC URL, user id, password etc).
- We can then autowire/inject the JPA Entity Manager into our DAO.

- Overall flow - $DAO ⇄ Entity Manager ⇄ Data Source ⇄ Database$

- `JpaRepository` -
  - Sprint Data JPA has a `JpaRepository` interface.
  - Provides JPA database access with minimal coding.

- `EntityManager` vs `JpaRepository` use-cases -
  - `EntityManager` - 
    - When you need _low-level control_ over the database operations and want to write custom queries.
    - Provides low-level access to JPA and work directly with JPA entities.
    - Complex queries that required advanced features such as native SQL queries or stored procedure call.s
    - When you have custom requirements that are not easily handled by higher-level abstractions.
  - `JpaRepository` -
    - When you need _high-level of abstraction_.
    - Provides commonly used CRUD operations out of the box, reducing the amount of code need to write.
    - Additional features such as pagination, sorting etc.
    - Generate queries based on method names.
    - Can also create custom queries using `@Query`.

## Saving Objects

- Define DAO interface -
```
public interface StudentDAO {
  void save(Student theStudent);
}
```

- Define DAO implementation -
```
@Repository
public class StudentDAOImpl implements StudentDAO {
  private EntityManager entityManager;

  @Autowired
  public StudentDAOImpl(EntityManager theEntityManager) {
    entityManager = theEntityManager;
  }

  @Override
  @Transactional
  public void save(Student theStudent) {
    entityManager.persist(theStudent);
  }
}
```

- `@Repository` -
  - `import org.springframework.stereotype.Repository`
  - Specialized annotation for repositories.
  - Another sub-annotation of `@Component`, just like `@RestController` - hence supports component scanning.
  - Translates JDBC exceptions.
- `@Transactional` -
  - `import org.springframework.transaction.annotation.Transactional`
  - Automatically begins and ends a transaction.

## Retrieving Objects

- By primary key -
```
@Override
public Student findById(Integer id) {
  Student student = entityManager.find(Student.class, id);
}
```

- If no student is found with `id` in primary key, then it returns `null`.

### JPA Query Language (JPQL)
  
- Query language for retrieving objects.
- Similar in concepts to SQL - `where`, `like`, `order by`, `join` etc.
- However, JPQL is based on entity name and entity fields.
- Example -
```
TypedQuery<Student> query = entityManager.createQuery("FROM Student WHERE lastName='Doe'", Student.class);
List<Student> students = query.getResultList();
```

- Note that in `FROM Student`, `Student` is the entity class name, not the table name. Similarly, `lastName` is the field name, not table column name.

- __Named Parameters__ -
```
public List<Student> findByLastName(String lastName) {
  TypedQuery<Student> query = entityManager.createQuery(
                                          "FROM Student WHERE lastName=:theData", Student.class);
  query.setParameter("theData", lastName);
  return query.getResultList();
}
```

- __`select` clause__ -
  - We did not need to specify `select` clause because the Hibernate implementation is lenient and allows Hibernate Query Language (HQL).
  - For strict JPQL, the `select` clause is required -
  ```
  TypedQuery<Student> query = entityManager.createQuery("SELECT s FROM Student", Student.class);
  ```
  
  - `s` is an alias that provides a reference to the `Student` entity object. It is useful when you have complex queries -
   ```
  entityManager.createQuery("SELECT s FROM Student s WHERE s.name LIKE :theData", Student.class);
  ```

## Updating Objects

- Manual query - useful for updating multiple records -
```
int numRowsUpdated = entityManager.createQuery("UPDATE Student SET lastName='Tester'")
                                  .executeUpdate();
```

- DAO implementation -
```
@Override
@Transactional
public void update(Student theStudent) {
  entityManager.merge(theStudent);
}
```

## Deleting Objects

- Manual query - useful for deleting multiple records -
```
int numRowsDeleted = entityManager.createQuery("DELETE FROM Student WHERE lastName='Doe'")
                                  .executeUpdate(); 
```

- DAO implementation -
```
@Override
@Transactional
public void delete(Student theStudent) {
  Student theStudent = entityManager.find(Student.class, id);
  entityManager.remove(theStudent);
}
```

## Creating Database Tables

- JPA / Hibernate create tables based on Java code with JPA / Hiberante annotations.
- Useful for development and testing.

- in `application.properties` - add `spring.jpa.hibernate.ddl-auto=create`

| Property Value | Description                                                                              | 
|----------------|------------------------------------------------------------------------------------------|
| `none`         | No action will be performed                                                              |
| `create`       | Database tables are dropped followed by database tables creation                         |
| `create-drop`  | Database tables are dropped on application shutdown followed by database tables creation |
| `validate`     | Validate the database tables schema                                                      |
| `update`       | Update the database tables schema                                                        |

