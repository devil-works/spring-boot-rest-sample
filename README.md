Spring Boot と MySQL 使用してRESTfulな CRUD（Create, Read, Update, Delete）機能をもつエンドポイント 実装したときのメモです。

ブラウザからエンドポイントを利用して最後におまけで

前提
・Eclipse はインストール済みであること。
[Java 学習メモ 【環境構築編】2022 年 12 月版 Eclipse のインストール方法](https://devil-code.com/blogs/java/eclipse/)

・STS（Spring Tool Suite4）はダウンロードしてインストール済みであること。
・MySQL はインストール済みであること。


**プロジェクトの構成**
Eclipse（Eclipse IDE for Enterprise Java and Web Developers）
Spring Boot Webアプリケーションフレームワーク
Maven ビルドツール
MySQLデータベース


**最終的なディレクトリ構成**
```
- src
  |- main
     |- java
        |- com.sample
          |- model
                └ User.java
          |- repository
                └ UserRepository.java
          |- controller
                └ UserController.java
          |- service  
                └ UserService.java
     |- resources
        |- application.properties
        |- templates
                └ user.html
        |- static
```


## 1. Spring Boot プロジェクトの作成

STS を起動し、新しい Spring Boot プロジェクトを作成します。Spring Initializr を使用して、必要な依存関係を含むプロジェクトを作成します。

Eclipse のメニューから「File」→「New」→「Other」→「Spring Boot」→「Spring Starter Project」を選択します。

![ウィザードの選択](image-4.png)

プロジェクト名は crud としてビルドツールは Maven を選択しました。

![プロジェクトの作成](image-7.png)


![依存関係](image-6.png)

WEB - Spring Web
SQL - Spring Data JPA, MySQL Driver
~~Template Engines - Thymeleaf~~

## 2.MySQL データベースのセットアップ
データベースを作成します。
```sql
create database sampledb;
use sampledb;
```

テーブルを作成します。
| カラム名 | データ型     | 制約        |
| :------- | :----------- | :---------- |
| id       | INT          | PRIMARY KEY |
| name     | VARCHAR(100) | NOT NULL    |
| email    | VARCHAR(100) | NOT NULL    |
| age      | INT          |             |
| address  | VARCHAR(200) |             |

```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL,
    age INT,
    address VARCHAR(200)
);
```

サンプルデータを作成します。
```sql
INSERT INTO users (name, email, age, address)
VALUES
    ('John Doe', 'john.doe@example.com', 30, '123 Main Street'),
    ('Jane Smith', 'jane.smith@example.com', 25, '456 Elm Avenue'),
    ('Mike Johnson', 'mike.johnson@example.com', 40, '789 Oak Road');
```

## 3. データモデルを作成
MySQLのテーブルとマッピングするエンティティクラスを作成します。JPAアノテーションを使用して、エンティティクラスをデータベーステーブルにマッピングします。
com.sample.modelパッケージを作成してその下にUser.javaクラスを作成します。

```
- src
  |- main
     |- java
        |- com.sample
          |- model    <-- 新しく作成するパッケージ
                └ User.java    <-- 新しく作成するクラス
     |- resources
        |- application.properties
        |- templates
        |- static
```



User.java

※バージョンによってはjakartaがjavaxかもしれませんので注意！
```java
import jakarta.persistence.*;

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "email", nullable = false)
    private String email;

    @Column(name = "age")
    private Integer age;

    @Column(name = "address")
    private String address;

    // Getter and Setter methods

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }
}
```

## データベース接続の設定
application.propertiesを使用して、MySQLデータベースの接続情報を設定します。

```
- src
  |- main
     |- java
        |- com.sample
          |- model
     |- resources
        |- application.properties    <-- これ
        |- templates
        |- static
```

```
# MySQLデータベース接続設定
spring.datasource.url=jdbc:mysql://localhost:3306/sample?useSSL=false
spring.datasource.username=your_mysql_username
spring.datasource.password=your_mysql_password

# JPA設定
spring.jpa.hibernate.ddl-auto=create

```

## リポジトリの作成
Spring Data JPAを使用して、データベースへのアクセスを行うためのリポジトリクラスを作成します
```
- src
  |- main
     |- java
        |- com.sample
          |- model
          |- repository    <-- 新しく作成するパッケージ
                └ UserRepository.java    <-- 新しく作成するクラス
     |- resources
        |- application.properties
        |- templates
        |- static
```

UserRepository.java
```java
package com.sample.repository;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import com.sample.model.User;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // 任意のカスタムクエリやメソッドを定義することができます
}
```

## コントローラーとサービスの実装
エンティティに対するCRUD操作を実行するためのコントローラーとサービスクラスを作成します。
コントローラークラスがエンドポイントを定義し、HTTPリクエストを受け取って対応するサービスメソッドを呼び出します。
サービスクラスはデータベースに対するCRUD操作を提供し、UserRepositoryを使用してデータベースにアクセスします。

```
- src
  |- main
     |- java
        |- com.sample
          |- model
          |- repository
          |- controller    <-- 新しく作成するパッケージ
                └ UserController.java    <-- 新しく作成するクラス
          |- service       <-- 新しく作成するパッケージ
                └ UserService.java    <-- 新しく作成するクラス
     |- resources
        |- application.properties
        |- templates
        |- static
```

UserController（コントローラークラス）の実装例
```
package com.sample.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import com.sample.model.User;
import com.sample.service.UserService;

@RestController
@RequestMapping("/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUserById(id);
    }

    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.saveUser(user);
    }

    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody User user) {
        return userService.updateUser(id, user);
    }

    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
    }
}
```

UserService（サービスクラス）の実装例
```
package com.sample.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import com.sample.model.User;
import com.sample.repository.UserRepository;

@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public User getUserById(Long id) {
        return userRepository.findById(id).orElse(null);
    }

    public User saveUser(User user) {
        return userRepository.save(user);
    }

    public User updateUser(Long id, User user) {
        user.setId(id);
        return userRepository.save(user);
    }

    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
}
```

これでエンドポイントは完成です。



## フロントエンドからJavaScriptでエンドポイントを利用してデータの操作をしてみる

```
- src
  |- main
     |- java
        |- com.sample
          |- model
          |- repository
          |- controller
          |- service
     |- resources
        |- application.properties
        |- templates
        |- static
                └ index.html    <-- 新しく作成するファイル
```


**Read**
index.html
```html
<!DOCTYPE html>
<html>
  <head>
    <title>User List</title>
  </head>
  <body>
    <h1>User List</h1>
    <table>
      <tr id="th"></tr>
      <tr id="td"></tr>
    </table>
    <script>
      window.onload = (event) => {
        fetch('http://localhost:8080/users/1')
			.then(response => response.json())
			.then(data => {
			    const th = document.getElementById('th');
			    const td = document.getElementById('td');

			    for (var key in data) {
			    	th.insertAdjacentHTML('beforeend', `<th>${key}</th>`);
			    	td.insertAdjacentHTML('beforeend', `<th>${data[key]}</th>`);
			    }
			})
			.catch(error => console.error('Error:', error));
      };
    </script>
  </body>
</html>
```


EclipseでSpring Bootアプリケーションを実行し、ブラウザからアクセスしてMySQLデータベースとのCRUD操作をテストします。

Project Explorerでプロジェクトを右クリック「Run As」→「Spring Boot App」を選択します。
