---
title: Declaring Models
layout: page
---

## Declaring Models

Models are normal structs with basic Go types, including support for type pointers and aliases. Custom types are supported that implement the [Scanner](https://pkg.go.dev/database/sql/?tab=doc#Scanner) and [Valuer](https://pkg.go.dev/database/sql/driver#Valuer) interfaces.

For Example:

```go
type User struct {
  ID           uint
  Name         string
  Email        *string
  Age          uint8
  Birthday     *time.Time
  MemberNumber sql.NullString
  ActivatedAt  sql.NullTime
  CreatedAt    time.Time
  UpdatedAt    time.Time
}
```

## Conventions

GORM prefers convention to configuration. By default, GORM uses a field named `ID` as the primary key, pluralize struct names in `snake_case` as the table name, `snake_case` fields as column names, and uses `CreatedAt` and `UpdatedAt` fields to track creating/updating time.

If you follow the conventions adopted by GORM, you will require very little configuration or additional code. If the conventions don't match your requirements, [GORM allows you to configure them](conventions.html).

## gorm.Model

GORM defines a `gorm.Model` struct, which includes the fields `ID`, `CreatedAt`, `UpdatedAt`, `DeletedAt`.

```go
// gorm.Model definition
type Model struct {
  ID        uint           `gorm:"primaryKey"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

You can embed the Model into your struct to include those fields. Refer to the [Embedded Struct](#embedded_struct) section for more info.

## Advanced

### <span id="field_permission">Field-Level Permission</span>

Exported fields have all permissions when performing CRUD operations with GORM, and GORM allows you to change the field-level permissions with struct tags. You can set a field to be read-only, write-only, create-only, update-only or ignored.

{% note warn %}
**NOTE** Ignored fields won't be created when using GORM Migrator to create a table.
{% endnote %}

```go
type User struct {
  Name string `gorm:"<-:create"` // allow read and create
  Name string `gorm:"<-:update"` // allow read and update
  Name string `gorm:"<-"`        // allow read and write (create and update)
  Name string `gorm:"<-:false"`  // allow read, disable write permission
  Name string `gorm:"->"`        // readonly (disable write permission unless it configured )
  Name string `gorm:"->;<-:create"` // allow read and create
  Name string `gorm:"->:false;<-:create"` // createonly (disabled read from db)
  Name string `gorm:"-"`  // ignore this field when write and read with struct
  Name string `gorm:"migration"` // // ignore this field when migration
}
```

### <name id="time_tracking">Creation/Updated Time/Unix (milli/nano) Seconds Tracking</span>

GORM uses the `CreatedAt` and `UpdatedAt` fields to track creation/updated time. GORM will set the [current time](gorm_config.html#now_func) when creating or updating an object if the fields are defined.

To use fields with a different name, you can configure those fields with the tag `autoCreateTime` or `autoUpdateTime` as seen below.

If you prefer to save UNIX (milli/nano) seconds instead of Go's time.Time struct, you can simply change the field's data type from `time.Time` to `int`.

```go
type User struct {
  CreatedAt time.Time // Set to current time if it is zero on creating
  UpdatedAt int       // Set to current unix seconds on updating or if it is zero on creating
  Updated   int64 `gorm:"autoUpdateTime:nano"` // Use unix nano seconds as updating time
  Updated   int64 `gorm:"autoUpdateTime:milli"`// Use unix milli seconds as updating time
  Created   int64 `gorm:"autoCreateTime"`      // Use unix seconds as creating time
}
```

### <span id="embedded_struct">Embedded Structs</span>

For anonymous fields, GORM will include its fields into its parent struct, for example:

```go
type User struct {
  gorm.Model
  Name string
}
// results in...
type User struct {
  ID        uint           `gorm:"primaryKey"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt gorm.DeletedAt `gorm:"index"`
  Name string
}
```

For a normal struct field, you can embed it with the tag `embedded`, for example:

```go
type Author struct {
	Name  string
	Email string
}

type Blog struct {
  ID      int
  Author  Author `gorm:"embedded"`
  Upvotes int32
}
// results in...
type Blog struct {
  ID    int64
	Name  string
	Email string
  Upvotes  int32
}
```

You can also use the tag `embeddedPrefix` to add a prefix to the embedded fields' db name, for example:

```go
type Blog struct {
  ID      int
  Author  Author `gorm:"embedded;embeddedPrefix:author_"`
  Upvotes int32
}
// results in...
type Blog struct {
  ID          int64
	AuthorName  string
	AuthorEmail string
  Upvotes     int32
}
```


### <span id="tags">Field Tags</span>

Struct tags are optional when declaring models. They allow you to tailor the database model to your liking. Multiple tags can be used on one field. Refer to the Golang spec on [Struct Types](https://go.dev/ref/spec#Struct_types) for more info.

Tags are case-insensitive, however, `camelCase` is preferred.
GORM supports the following tags:

| Tag Name       | Description                                                            |
| ---            | ---                                                                    |
| column         | column db name                                                  |
| type           | column data type, prefer to use compatible general type, e.g: bool, int, uint, float, string, time, bytes, which works for all databases, and can be used with other tags together, like `not null`, `size`, `autoIncrement`... specified database data type like `varbinary(8)` also supported, when using specified database data type, it needs to be a full database data type, for example: `MEDIUMINT UNSIGNED NOT NULL AUTO_INCREMENT` |
| size           | specifies column data size/length, e.g: `size:256`                                                  |
| primaryKey     | specifies column as primary key                                        |
| unique         | specifies column as unique                                             |
| default        | specifies column default value                                         |
| precision      | specifies column precision                                             |
| scale          | specifies column scale                                                 |
| not null       | specifies column as NOT NULL                                           |
| autoIncrement  | specifies column auto incrementable                                    |
| autoIncrementIncrement  | auto increment step, controls the interval between successive column values |
| embedded       | embed the field                                                        |
| embeddedPrefix | column name prefix for embedded fields                                 |
| autoCreateTime | track current time when creating, for `int` fields, it will track unix seconds, use value `nano`/`milli` to track unix nano/milli seconds, e.g: `autoCreateTime:nano` |
| autoUpdateTime | track current time when creating/updating, for `int` fields, it will track unix seconds, use value `nano`/`milli` to track unix nano/milli seconds, e.g: `autoUpdateTime:milli` |
| index          | create index with options, use same name for multiple fields creates composite indexes, refer [Indexes](indexes.html) for details |
| uniqueIndex    | same as `index`, but create unique index                              |
| check          | creates check constraint, eg: `check:age > 13`, refer [Constraints](constraints.html) |
| <-             | set field's write permission, `<-:create` create-only field, `<-:update` update-only field, `<-:false` no write permission, `<-` create and update permission |
| ->             | set field's read permission, `->:false` no read permission             |
| -              | ignore this field, `-` no read/write permission                       |
| comment        | add comment for field when migration                                  |

### Association Tags

GORM allows configuring foreign keys, constraints, and many2many assocaiations through tags. Refer to the [Associations section](associations.html#tags) for details.
