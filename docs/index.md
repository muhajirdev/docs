---
template: main.html
---

# Getting started

go-pg requires latest Go version with
[Modules](https://github.com/golang/go/wiki/Modules) support and uses import
versioning. So make sure to initialize a Go module:

```shell
go mod init github.com/my/repo
```

and then install go-pg:

```shell
go get github.com/go-pg/pg/v10
```

To connect to a database use:

```go
db := pg.Connect(&pg.Options{
    Addr:     ":5432",
    User:     "user",
    Password: "pass",
    Database: "db_name",
})
```

Another popular way is using a connection string:

```go
opt, err := pg.ParseURL("postgres://user:pass@localhost:5432/db_name")
if err != nil {
   panic(err)
}

db := pg.Connect(opt)
```

After that you can start executing queries:

```go
// Check if connection credentials are valid and PostgreSQL is up and running.
if err := db.Ping(); err != nil {
    panic(err)
}
```

Following example is more complex and demonstrates how to connect, create
schema, insert, and select data:

```go
package main

import (
    "fmt"

    "github.com/go-pg/pg/v10"
    "github.com/go-pg/pg/v10/orm"
)

func main() {
    db := pg.Connect(&pg.Options{
        Addr:     ":5432",
        User:     "postgres",
        Password: "",
    })
    defer db.Close()

    err := createSchema(db)
    if err != nil {
        panic(err)
    }

    user1 := &User{
        Name:   "admin",
        Emails: []string{"admin1@admin", "admin2@admin"},
    }
    err = db.Insert(user1)
    if err != nil {
        panic(err)
    }

    err = db.Insert(&User{
        Name:   "root",
        Emails: []string{"root1@root", "root2@root"},
    })
    if err != nil {
        panic(err)
    }

    story1 := &Story{
        Title:    "Cool story",
        AuthorID: user1.ID,
    }
    err = db.Insert(story1)
    if err != nil {
        panic(err)
    }

    // Select user by primary key.
    user := &User{ID: user1.ID}
    err = db.Select(user)
    if err != nil {
        panic(err)
    }

    // Select all users.
    var users []User
    err = db.Model(&users).Select()
    if err != nil {
        panic(err)
    }

    // Select story and associated author in one query.
    story := new(Story)
    err = db.Model(story).
        Relation("Author").
        Where("story.id = ?", story1.ID).
        Select()
    if err != nil {
        panic(err)
    }

    fmt.Println(user)
    fmt.Println(users)
    fmt.Println(story)
    // Output: User<1 admin [admin1@admin admin2@admin]>
    // [User<1 admin [admin1@admin admin2@admin]> User<2 root [root1@root root2@root]>]
    // Story<1 Cool story User<1 admin [admin1@admin admin2@admin]>>
}

type User struct {
    ID     int64
    Name   string
    Emails []string
}

func (u *User) String() string {
    return fmt.Sprintf("User<%d %s %v>", u.ID, u.Name, u.Emails)
}

type Story struct {
    ID       int64
    Title    string
    AuthorID int64
    Author   *User
}

func (s *Story) String() string {
    return fmt.Sprintf("Story<%d %s %s>", s.ID, s.Title, s.Author)
}

// createSchema creates database schema for User and Story models.
func createSchema(db *pg.DB) error {
    models := []interface{}{
        (*User)(nil),
        (*Story)(nil),
    }

    for _, model := range models {
        err := db.CreateTable(model, &orm.CreateTableOptions{
            Temp: true, // temp table
        })
        if err != nil {
            return err
        }
    }
    return nil
}
```
