---
title: ORM - Skinny Framework
---

## ORM

<hr/>
### Skinny ORM

Skinny provides you Skinny-ORM as the default O/R mapper, which is built with [ScalikeJDBC](https://github.com/scalikejdbc/scalikejdbc).

![Logo](images/scalikejdbc.png)

<hr/>
#### Simple Mapper

Skinny-ORM is much powerful, so you don't need to write much code. Your first model class and companion are here.

```java
case class Member(id: Long, name: String, createdAt: DateTime)

object Member extends SkinnyCRUDMapper[Member] {
  // only define ResultSet extractor at minimum
  override def extract(rs: WrappedResultSet, n: ResultName[Member]) = new Member(
    id = rs.long(n.id),
    name = rs.string(n.name),
    createdAt = rs.dateTime(n.createdAt)
  )
}
```

That's all! Now you can use the following APIs.

```java
Member.withAlias { m => // or "val m = Member.defaultAlias"
  // find by primary key
  val member: Option[Member] = Member.findById(123)
  val member: Option[Member] = Member.where('id -> 123).apply().headOption
  val members: List[Member] = Member.where('id -> Seq(123, 234, 345)).apply()
  // find many
  val members: List[Member] = Member.findAll()
  val groupMembers = Member.where('groupName -> "Scala Users", 'deleted -> false).apply()
  // count
  val allCount: Long = Member.countAll()
  val count = Member.countBy(sqls.isNotNull(m.deletedAt).and.eq(m.countryId, 123))
  val count = Member.where('deletedAt -> None, 'countryId -> 123).count.apply()

  // create with stong parameters
  val params = Map("name" -> "Bob")
  val id = Member.createWithPermittedAttributes(
    params.permit("name" -> ParamType.String))
  // create with unsafe parameters
  Member.createWithAttributes(
    'id -> 123,
    'name -> "Chris",
    'createdAt -> DateTime.now
  )

  // update with strong parameters
  Member.updateById(123).withAttributes(params.permit("name" -> ParamType.String))
  // update with unsafe parameters
  Member.updateById(123).withAttributes('name -> "Alice")

  // delete
  Member.deleteById(234)
}
```

<hr/>
#### Relationships

If you need to join other tables, just add `belongsTo`, `hasOne` or `hasMany` (`hasManyThrough`) to the companion.

```java
class Member(id: Long, name: String, companyId: Long,
  company: Option[Company] = None, skills: Seq[Skill] = Nil)

object Member extends SkinnyCRUDMapper[Member] {

  // If byDefault is called, this join condition is enabled by default
  belongsTo[Company](Company, (m, c) => m.copy(company = Some(c))).byDefault

  val skills = hasManyThrough[Skill](
    MemberSkill, Skill, (m, skills) => m.copy(skills = skills))
}

Member.findById(123) // without skills
Member.joins(Member.skills).findById(123) // with skills
```

<hr/>
#### Eager Loading

You can call `includes` for eager loading. But nested entities's eager loading is not supported yet.

```java
object Member extends SkinnyCRUDMapper[Member] {
  val skills =
    hasManyThrough[Skill](
      MemberSkill, Skill, (m, skills) => m.copy(skills = skills)
    ).includes[Skill]((ms, skills) => ms.map { m =>
      m.copy(kills = skills.filter(_.memberId.exists(_ == m.id)))
    })
}

Member.includes(Member.skills).findById(123) // with skills
```

<hr/>
#### Adding Methods

If you need to add methods, just write methods that use ScalikeJDBC' APIs directly.

```java
object Member extends SkinnyCRUDMapper[Member] {
  val m = defaultAlias
  def findByGroupId(groupId: Long)(implicit s: DBSession = autoSession): List[Member] =
    withSQL { select.from(Member as m).where.eq(m.groupId, groupId) }
      .map(apply(m)).list.apply()
}
```

If you're using ORM with Skinny Framework,
[TxPerRequestFilter](https://github.com/skinny-framework/skinny-framework/blob/master/orm/src/main/scala/skinny/servlet/TxPerRequestFilter.scala) can simplify your applciations.

```java
// src/main/scala/ScalatraBootstrap.scala
class ScalatraBootstrap exntends SkinnyLifeCycle {
  override def initSkinnyApp(ctx: ServletContext) {
    ctx.mount(new TxPerRequestFilter, "/*")
  }
}
```

And then your ORM models can retrieve current DB session (per request), so you don't need to pass `DBSession` value as an implicit parameter in each method.

```java
  def findByGroupId(groupId: Long): List[Member] =
    withSQL { select.from(Member as m).where.eq(m.groupId, groupId) }
      .map(apply(m)).list.apply()
```


<hr/>
#### Timestamps

`timetamps` from ActiveRecord is available as the `TimestampsFeature` trait.

```java
class Member(id: Long, name: String,
  createdAt: DateTime,
  updatedAt: Option[DateTime] = None)

object Member extends SkinnyCRUDMapper[Member] with TimestampsFeature[Member]
// created_at timestamp not null, updated_at timestamp
```

<hr/>
#### Soft Deletion

Soft delete support is also available.

```java
object Member extends SkinnyCRUDMapper[Member]
  with SoftDeleteWithTimestamp[Member]
// deleted_at timestamp
```

<hr/>
#### Optimistic Lock

Furthermore, optimistic lock is also available.

```java
object Member extends SkinnyCRUDMapper[Member]
  with OptimisticLockWithVersionFeature[Member]
// lock_version bigint
```

<hr/>
#### More Examples

You can see more examples here:

[orm/src/test/scala/skinny/orm/models.scala](https://github.com/skinny-framework/skinny-framework/blob/master/orm/src/test/scala/skinny/orm/models.scala)

[orm/src/test/scala/skinny/orm/SkinnyORMSpec.scala](https://github.com/skinny-framework/skinny-framework/blob/master/orm/src/test/scala/skinny/orm/SkinnyORMSpec.scala)
