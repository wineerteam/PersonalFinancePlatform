# Personal Finance Platform

## Overview

The Personal Finance Platform is a Java-based application designed to help users manage their finances effectively. It provides functionalities for tracking expenses, setting budgets, and generating financial reports. The application is structured to support different user roles, including administrators, financial advisors, and regular users.

## Project Structure

```
PersonalFinancePlatform
├── src
│   ├── Main.java                # Entry point of the application
│   ├── Database.java            # Handles database connection and queries
│   ├── AdminDashboard.java      # GUI for admin functionalities
│   ├── AdvisorDashboard.java    # GUI for financial advisors
│   ├── UserDashboard.java       # GUI for regular users
│   └── Models
│       ├── User.java            # Represents a user in the system
│       ├── Expense.java         # Represents an expense
│       └── Budget.java          # Represents a budget
├── libs
│   └── sqlite-jdbc-3.36.0.3.jar # SQLite JDBC driver
├── resources
│   ├── icons                    # Directory for GUI icons
│   └── css
│       └── theme.css            # Custom styling for the GUI
├── database
│   └── finance.db               # SQLite database file
├── launch.json                  # Launch configuration for development
└── README.md                    # Project documentation
```

## Setup Instructions

1. **Clone the Repository**
   Clone the repository to your local machine using:

   ```
   git clone <repository-url>
   ```

2. **Install Java**
   Ensure that you have Java Development Kit (JDK) installed on your machine. You can download it from the [official Oracle website](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html).

3. **Add SQLite JDBC Driver**
   The SQLite JDBC driver is included in the `libs` directory. Ensure that the path to `sqlite-jdbc-3.36.0.3.jar` is correctly referenced in your project settings.

4. **Database Setup**
   The SQLite database file `finance.db` will be automatically created when the application runs for the first time. Ensure that the application has write permissions to the `database` directory.

5. **Run the Application**
   You can run the application by executing the `Main.java` file. This will initialize the application and open the GUI.

## Usage

- **Admin Dashboard**: Manage users, view reports, and perform administrative tasks.
- **Advisor Dashboard**: Manage client portfolios and provide financial advice.
- **User Dashboard**: Track expenses, set budgets, and view financial summaries.

## Contributing

Contributions are welcome! Please submit a pull request or open an issue for any enhancements or bug fixes.

## License

This project is licensed under the MIT License. See the LICENSE file for more details.

## Build Configuration

The project uses Gradle as the build tool. Below is the `build.gradle` configuration:

```groovy
plugins {
    id 'org.springframework.boot' version '3.1.6'
    id 'io.spring.dependency-management' version '1.1.0'
    id 'java'
}

group = 'com.example'
version = '1.0.0'
sourceCompatibility = '17'

repositories { mavenCentral() }

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    implementation 'org.xerial:sqlite-jdbc:3.36.0.3'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') { useJUnitPlatform() }
```

rootProject.name = 'PersonalFinancePlatform-web'

package com.example.pfp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import java.io.File;

@SpringBootApplication
public class PersonalFinancePlatformApplication {
    public static void main(String[] args) { SpringApplication.run(PersonalFinancePlatformApplication.class, args); }

    // Ensure database folder exists before DataSource init
    @Bean
    public boolean ensureDbDir() {
        File db = new File("database");
        if (!db.exists()) db.mkdirs();
        return true;
    }
}

package com.example.pfp.service;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import javax.annotation.PostConstruct;
import java.util.List;
import java.util.Map;

@Service
public class DatabaseService {
    private final JdbcTemplate jdbc;
    public DatabaseService(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    @PostConstruct
    public void init() {
        jdbc.execute("PRAGMA foreign_keys = ON;");
        jdbc.execute("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE, role TEXT);");
        jdbc.execute("CREATE TABLE IF NOT EXISTS expenses (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, amount REAL, category TEXT, description TEXT, date TEXT, FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE);");
        jdbc.execute("CREATE TABLE IF NOT EXISTS budgets (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, amount REAL, period TEXT, FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE);");
    }

    public long ensureUser(String username, String role) {
        if (username == null || username.trim().isEmpty()) return -1;
        username = username.trim();
        role = (role == null || role.trim().isEmpty()) ? "user" : role.trim();
        List<Map<String,Object>> rows = jdbc.queryForList("SELECT id FROM users WHERE username = ?", username);
        if (!rows.isEmpty()) return ((Number)rows.get(0).get("id")).longValue();
        jdbc.update("INSERT INTO users(username, role) VALUES(?,?)", username, role);
        return jdbc.queryForObject("SELECT id FROM users WHERE username = ?", Long.class, username);
    }

    public List<Map<String,Object>> listUsers() { return jdbc.queryForList("SELECT id, username, role FROM users"); }

    public boolean addExpense(long userId, double amount, String category, String desc, String date) {
        return jdbc.update("INSERT INTO expenses(user_id,amount,category,description,date) VALUES(?,?,?,?,?)", userId, amount, category, desc, date) > 0;
    }
    public List<Map<String,Object>> listExpenses(long userId) {
        return jdbc.queryForList("SELECT id, amount, category, description, date FROM expenses WHERE user_id = ? ORDER BY date DESC", userId);
    }

    public boolean addBudget(long userId, double amount, String period) {
        return jdbc.update("INSERT INTO budgets(user_id,amount,period) VALUES(?,?,?)", userId, amount, period) > 0;
    }
    public List<Map<String,Object>> listBudgets(long userId) {
        return jdbc.queryForList("SELECT id, amount, period FROM budgets WHERE user_id = ?", userId);
    }
}

package com.example.pfp.controller;

import com.example.pfp.service.DatabaseService;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.util.List;
import java.util.Map;

@Controller
public class HomeController {
    private final DatabaseService db;
    public HomeController(DatabaseService db){ this.db = db; }

    @GetMapping("/")
    public String index() { return "index"; }

    // Admin
    @GetMapping("/admin")
    public String admin(Model m){
        List<Map<String,Object>> users = db.listUsers();
        m.addAttribute("users", users);
        return "admin";
    }
    @PostMapping("/admin/user")
    public String addUser(@RequestParam String username, @RequestParam String role){ db.ensureUser(username, role); return "redirect:/admin"; }

    // Advisor
    @GetMapping("/advisor")
    public String advisor(Model m){
        m.addAttribute("users", db.listUsers());
        return "advisor";
    }
    @GetMapping("/advisor/user/{id}")
    public String advisorUser(@PathVariable long id, Model m){
        m.addAttribute("userId", id);
        m.addAttribute("expenses", db.listExpenses(id));
        return "advisor_user";
    }

    // User
    @GetMapping("/user/{id}")
    public String userDashboard(@PathVariable long id, Model m){
        m.addAttribute("userId", id);
        m.addAttribute("expenses", db.listExpenses(id));
        m.addAttribute("budgets", db.listBudgets(id));
        return "user";
    }

    @PostMapping("/user/{id}/expense")
    public String addExpense(@PathVariable long id, @RequestParam double amount, @RequestParam String category, @RequestParam(required=false) String description){
        db.addExpense(id, amount, category, description == null ? "" : description, LocalDate.now().toString());
        return "redirect:/user/" + id;
    }

    @PostMapping("/user/{id}/budget")
    public String addBudget(@PathVariable long id, @RequestParam double amount, @RequestParam String period){
        db.addBudget(id, amount, period);
        return "redirect:/user/" + id;
    }
}

package com.example.pfp.controller;

import com.example.pfp.service.DatabaseService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ApiController {
    private final DatabaseService db;
    public ApiController(DatabaseService db){ this.db = db; }

    @GetMapping("/api/open")
    public String openUser(@RequestParam String username){
        long id = db.ensureUser(username, "user");
        return Long.toString(id);
    }
}
spring.datasource.url=jdbc:sqlite:database/finance.db
spring.datasource.driver-class-name=org.sqlite.JDBC
# Ensure Spring will initialize resources if you later use schema/data SQL files
spring.sql.init.mode=always
spring.thymeleaf.cache=false
server.port=8080
logging.level.root=INFO

<!DOCTYPE html><html><head><meta charset="utf-8"><title>Admin</title></head><body>
<h2>Admin Dashboard</h2>
<form action="/admin/user" method="post">
  Username: <input name="username"> Role: <input name="role" value="user">
  <button type="submit">Add User</button>
</form>
<h3>Users</h3>
<table border="1"><tr><th>ID</th><th>Username</th><th>Role</th></tr>
  <th:block th:each="u : ${users}">
    <tr><td th:text="${u.id}"></td><td th:text="${u.username}"></td><td th:text="${u.role}"></td></tr>
  </th:block>
</table>
<a href="/">Back</a>
</body></html>

<!DOCTYPE html><html><head><meta charset="utf-8"><title>Advisor</title></head><body>
<h2>Advisor Dashboard</h2>
<ul>
  <li th:each="u: ${users}"><a th:href="@{'/advisor/user/' + ${u.id}}" th:text="${u.username + ' (' + u.role + ')'}"></a></li>
</ul>
<a href="/">Back</a>
</body></html>

<!DOCTYPE html><html><head><meta charset="utf-8"><title>Advisor - User</title></head><body>
<h2>Expenses for user id [[${userId}]]</h2>
<pre th:text="${#lists.size(expenses) == 0 ? 'No expenses' : ''}"></pre>
<ul>
  <li th:each="e: ${expenses}" th:text="${e.date + ' | ' + e.category + ' | ' + e.amount + ' | ' + e.description}"></li>
</ul>
<a href="/advisor">Back</a>
</body></html>

<!DOCTYPE html><html><head><meta charset="utf-8"><title>User</title></head><body>
<h2>User Dashboard - id [[${userId}]]</h2>

<h3>Add Expense</h3>
<form th:action="@{'/user/' + ${userId} + '/expense'}" method="post">
  Amount: <input name="amount" required> Category: <input name="category" required> Desc: <input name="description">
  <button type="submit">Add</button>
</form>

<h3>Expenses</h3>
<ul><li th:each="e:${expenses}" th:text="${e.date + ' | ' + e.category + ' | ' + e.amount + ' | ' + e.description}"></li></ul>

<h3>Add Budget</h3>
<form th:action="@{'/user/' + ${userId} + '/budget'}" method="post">
  Amount:<input name="amount" required> Period:<input name="period" required>
  <button type="submit">Add</button>
</form>

<h3>Budgets</h3>
<ul><li th:each="b:${budgets}" th:text="${b.period + ' : ' + b.amount}"></li></ul>

<a href="/">Back</a>
</body></html>

# PersonalFinancePlatform Project Structure & File Roles

This document explains the main files and their purposes in the project, organized by folder structure. Use this as a reference for understanding, development, and hosting.

---

## 1. Project Root

- **build.gradle**: Gradle build script. Manages dependencies (Spring Boot, PostgreSQL, etc.), plugins, and build tasks.
- **settings.gradle**: Gradle project settings.
- **README.md**: This documentation file.
- **launch.json**: VS Code debug configuration.

## 2. Database

- **database/finance.db**: (Legacy) SQLite database file. Not used in cloud hosting; replaced by PostgreSQL.
- **libs/sqlite-jdbc-3.36.0.3.jar**: (Legacy) SQLite JDBC driver. Not used in cloud hosting.

## 3. Resources

- **resources/icons/**: App icons and favicon.
- **resources/css/theme.css**: Custom CSS theme for UI.

## 4. Templates (UI)

Located in `src/main/resources/templates/`:
- **index.html**: Main dashboard landing page. Shows analytics, charts, and navigation for all roles.
- **user.html**: User dashboard. Allows users to add/view expenses and budgets. Integrates AI category suggestion.
- **admin.html**: Admin dashboard. Manage users and view user list.
- **advisor_user.html**: Advisor's view of a user's expenses.

## 5. Java Source Code

Located in `src/main/java/com/example/` and subfolders:

### Main Application
- **PersonalFinancePlatformApplication.java**: Spring Boot main entry point. Starts the web server.

### Controllers
- **controller/ApiController.java**: REST API endpoints for stats, user management, and AI features.
- **pfp/controller/HomeController.java**: Handles web page routing for Thymeleaf templates.

### Services
- **pfp/service/DatabaseService.java**: Main database logic using PostgreSQL. Handles CRUD for users, expenses, budgets, and demo data insertion.
- **service/StatsService.java**: (Legacy/compat) Analytics and stats logic (now migrated to PostgreSQL).
- **service/SimpleAICategoryService.java**: AI-powered category suggestion logic for expenses.
- **service/StatsServiceDisabled.java**: Placeholder/disabled service (not used).

### Models
- **Models/User.java**: User entity/model.
- **Models/Expense.java**: Expense entity/model.
- **Models/Budget.java**: Budget entity/model.

### Legacy
- **Database.java**: Old database logic for SQLite (not used in cloud hosting).
- **UserDashboard.java, AdminDashboard.java, AdvisorDashboard.java, Main.java**: Old UI logic (replaced by Thymeleaf templates).

---

## 6. Configuration

- **src/main/resources/application.properties**: Main Spring Boot and database configuration. Set for PostgreSQL and cloud hosting.

---

## 7. How the Code Works (Summary)

- **Spring Boot** starts the app and serves web pages and REST APIs.
- **Thymeleaf templates** render dynamic HTML for each user role.
- **DatabaseService** manages all data in PostgreSQL (users, expenses, budgets, demo data).
- **ApiController** provides REST endpoints for analytics, user management, and AI features.
- **SimpleAICategoryService** gives AI-based category suggestions for expenses.
- **StatsService** (legacy) provides analytics logic (now migrated to PostgreSQL).
- **Models** define the structure of users, expenses, and budgets.
- **application.properties** configures the database and server for local/public hosting.

---

## 8. Hosting

- For local: Use Gradle to run the app, connect to local PostgreSQL.
- For public: Deploy to Render/Railway, set environment variables for PostgreSQL.

---

For more details, see comments in each file or ask for a specific file explanation.

---

# (Legacy content below for reference)

# Personal Finance Platform

## Overview

The Personal Finance Platform is a Java-based application designed to help users manage their finances effectively. It provides functionalities for tracking expenses, setting budgets, and generating financial reports. The application is structured to support different user roles, including administrators, financial advisors, and regular users.

## Project Structure

```
PersonalFinancePlatform
├── src
│   ├── Main.java                # Entry point of the application
│   ├── Database.java            # Handles database connection and queries
│   ├── AdminDashboard.java      # GUI for admin functionalities
│   ├── AdvisorDashboard.java    # GUI for financial advisors
│   ├── UserDashboard.java       # GUI for regular users
│   └── Models
│       ├── User.java            # Represents a user in the system
│       ├── Expense.java         # Represents an expense
│       └── Budget.java          # Represents a budget
├── libs
│   └── sqlite-jdbc-3.36.0.3.jar # SQLite JDBC driver
├── resources
│   ├── icons                    # Directory for GUI icons
│   └── css
│       └── theme.css            # Custom styling for the GUI
├── database
│   └── finance.db               # SQLite database file
├── launch.json                  # Launch configuration for development
└── README.md                    # Project documentation
```

## Setup Instructions

1. **Clone the Repository**
   Clone the repository to your local machine using:

   ```
   git clone <repository-url>
   ```

2. **Install Java**
   Ensure that you have Java Development Kit (JDK) installed on your machine. You can download it from the [official Oracle website](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html).

3. **Add SQLite JDBC Driver**
   The SQLite JDBC driver is included in the `libs` directory. Ensure that the path to `sqlite-jdbc-3.36.0.3.jar` is correctly referenced in your project settings.

4. **Database Setup**
   The SQLite database file `finance.db` will be automatically created when the application runs for the first time. Ensure that the application has write permissions to the `database` directory.

5. **Run the Application**
   You can run the application by executing the `Main.java` file. This will initialize the application and open the GUI.

## Usage

- **Admin Dashboard**: Manage users, view reports, and perform administrative tasks.
- **Advisor Dashboard**: Manage client portfolios and provide financial advice.
- **User Dashboard**: Track expenses, set budgets, and view financial summaries.

## Contributing

Contributions are welcome! Please submit a pull request or open an issue for any enhancements or bug fixes.

## License

This project is licensed under the MIT License. See the LICENSE file for more details.

## Build Configuration

The project uses Gradle as the build tool. Below is the `build.gradle` configuration:

```groovy
plugins {
    id 'org.springframework.boot' version '3.1.6'
    id 'io.spring.dependency-management' version '1.1.0'
    id 'java'
}

group = 'com.example'
version = '1.0.0'
sourceCompatibility = '17'

repositories { mavenCentral() }

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    implementation 'org.xerial:sqlite-jdbc:3.36.0.3'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') { useJUnitPlatform() }
```

rootProject.name = 'PersonalFinancePlatform-web'

package com.example.pfp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import java.io.File;

@SpringBootApplication
public class PersonalFinancePlatformApplication {
    public static void main(String[] args) { SpringApplication.run(PersonalFinancePlatformApplication.class, args); }

    // Ensure database folder exists before DataSource init
    @Bean
    public boolean ensureDbDir() {
        File db = new File("database");
        if (!db.exists()) db.mkdirs();
        return true;
    }
}

package com.example.pfp.service;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import javax.annotation.PostConstruct;
import java.util.List;
import java.util.Map;

@Service
public class DatabaseService {
    private final JdbcTemplate jdbc;
    public DatabaseService(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    @PostConstruct
    public void init() {
        jdbc.execute("PRAGMA foreign_keys = ON;");
        jdbc.execute("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE, role TEXT);");
        jdbc.execute("CREATE TABLE IF NOT EXISTS expenses (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, amount REAL, category TEXT, description TEXT, date TEXT, FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE);");
        jdbc.execute("CREATE TABLE IF NOT EXISTS budgets (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, amount REAL, period TEXT, FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE);");
    }

    public long ensureUser(String username, String role) {
        if (username == null || username.trim().isEmpty()) return -1;
        username = username.trim();
        role = (role == null || role.trim().isEmpty()) ? "user" : role.trim();
        List<Map<String,Object>> rows = jdbc.queryForList("SELECT id FROM users WHERE username = ?", username);
        if (!rows.isEmpty()) return ((Number)rows.get(0).get("id")).longValue();
        jdbc.update("INSERT INTO users(username, role) VALUES(?,?)", username, role);
        return jdbc.queryForObject("SELECT id FROM users WHERE username = ?", Long.class, username);
    }

    public List<Map<String,Object>> listUsers() { return jdbc.queryForList("SELECT id, username, role FROM users"); }

    public boolean addExpense(long userId, double amount, String category, String desc, String date) {
        return jdbc.update("INSERT INTO expenses(user_id,amount,category,description,date) VALUES(?,?,?,?,?)", userId, amount, category, desc, date) > 0;
    }
    public List<Map<String,Object>> listExpenses(long userId) {
        return jdbc.queryForList("SELECT id, amount, category, description, date FROM expenses WHERE user_id = ? ORDER BY date DESC", userId);
    }

    public boolean addBudget(long userId, double amount, String period) {
        return jdbc.update("INSERT INTO budgets(user_id,amount,period) VALUES(?,?,?)", userId, amount, period) > 0;
    }
    public List<Map<String,Object>> listBudgets(long userId) {
        return jdbc.queryForList("SELECT id, amount, period FROM budgets WHERE user_id = ?", userId);
    }
}

package com.example.pfp.controller;

import com.example.pfp.service.DatabaseService;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.util.List;
import java.util.Map;

@Controller
public class HomeController {
    private final DatabaseService db;
    public HomeController(DatabaseService db){ this.db = db; }

    @GetMapping("/")
    public String index() { return "index"; }

    // Admin
    @GetMapping("/admin")
    public String admin(Model m){
        List<Map<String,Object>> users = db.listUsers();
        m.addAttribute("users", users);
        return "admin";
    }
    @PostMapping("/admin/user")
    public String addUser(@RequestParam String username, @RequestParam String role){ db.ensureUser(username, role); return "redirect:/admin"; }

    // Advisor
    @GetMapping("/advisor")
    public String advisor(Model m){
        m.addAttribute("users", db.listUsers());
        return "advisor";
    }
    @GetMapping("/advisor/user/{id}")
    public String advisorUser(@PathVariable long id, Model m){
        m.addAttribute("userId", id);
        m.addAttribute("expenses", db.listExpenses(id));
        return "advisor_user";
    }

    // User
    @GetMapping("/user/{id}")
    public String userDashboard(@PathVariable long id, Model m){
        m.addAttribute("userId", id);
        m.addAttribute("expenses", db.listExpenses(id));
        m.addAttribute("budgets", db.listBudgets(id));
        return "user";
    }

    @PostMapping("/user/{id}/expense")
    public String addExpense(@PathVariable long id, @RequestParam double amount, @RequestParam String category, @RequestParam(required=false) String description){
        db.addExpense(id, amount, category, description == null ? "" : description, LocalDate.now().toString());
        return "redirect:/user/" + id;
    }

    @PostMapping("/user/{id}/budget")
    public String addBudget(@PathVariable long id, @RequestParam double amount, @RequestParam String period){
        db.addBudget(id, amount, period);
        return "redirect:/user/" + id;
    }
}

package com.example.pfp.controller;

import com.example.pfp.service.DatabaseService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ApiController {
    private final DatabaseService db;
    public ApiController(DatabaseService db){ this.db = db; }

    @GetMapping("/api/open")
    public String openUser(@RequestParam String username){
        long id = db.ensureUser(username, "user");
        return Long.toString(id);
    }
}
spring.datasource.url=jdbc:sqlite:database/finance.db
spring.datasource.driver-class-name=org.sqlite.JDBC
# Ensure Spring will initialize resources if you later use schema/data SQL files
spring.sql.init.mode=always
spring.thymeleaf.cache=false
server.port=8080
logging.level.root=INFO

<!DOCTYPE html><html><head><meta charset="utf-8"><title>Admin</title></head><body>
<h2>Admin Dashboard</h2>
<form action="/admin/user" method="post">
  Username: <input name="username"> Role: <input name="role" value="user">
  <button type="submit">Add User</button>
</form>
<h3>Users</h3>
<table border="1"><tr><th>ID</th><th>Username</th><th>Role</th></tr>
  <th:block th:each="u : ${users}">
    <tr><td th:text="${u.id}"></td><td th:text="${u.username}"></td><td th:text="${u.role}"></td></tr>
  </th:block>
</table>
<a href="/">Back</a>
</body></html>

<!DOCTYPE html><html><head><meta charset="utf-8"><title>Advisor</title></head><body>
<h2>Advisor Dashboard</h2>
<ul>
  <li th:each="u: ${users}"><a th:href="@{'/advisor/user/' + ${u.id}}" th:text="${u.username + ' (' + u.role + ')'}"></a></li>
</ul>
<a href="/">Back</a>
</body></html>

<!DOCTYPE html><html><head><meta charset="utf-8"><title>Advisor - User</title></head><body>
<h2>Expenses for user id [[${userId}]]</h2>
<pre th:text="${#lists.size(expenses) == 0 ? 'No expenses' : ''}"></pre>
<ul>
  <li th:each="e: ${expenses}" th:text="${e.date + ' | ' + e.category + ' | ' + e.amount + ' | ' + e.description}"></li>
</ul>
<a href="/advisor">Back</a>
</body></html>

<!DOCTYPE html><html><head><meta charset="utf-8"><title>User</title></head><body>
<h2>User Dashboard - id [[${userId}]]</h2>

<h3>Add Expense</h3>
<form th:action="@{'/user/' + ${userId} + '/expense'}" method="post">
  Amount: <input name="amount" required> Category: <input name="category" required> Desc: <input name="description">
  <button type="submit">Add</button>
</form>

<h3>Expenses</h3>
<ul><li th:each="e:${expenses}" th:text="${e.date + ' | ' + e.category + ' | ' + e.amount + ' | ' + e.description}"></li></ul>

<h3>Add Budget</h3>
<form th:action="@{'/user/' + ${userId} + '/budget'}" method="post">
  Amount:<input name="amount" required> Period:<input name="period" required>
  <button type="submit">Add</button>
</form>

<h3>Budgets</h3>
<ul><li th:each="b:${budgets}" th:text="${b.period + ' : ' + b.amount}"></li></ul>

<a href="/">Back</a>
</body></html>

# PersonalFinancePlatform Project Structure & File Roles

This document explains the main files and their purposes in the project, organized by folder structure. Use this as a reference for understanding, development, and hosting.

---

## 1. Project Root

- **build.gradle**: Gradle build script. Manages dependencies (Spring Boot, PostgreSQL, etc.), plugins, and build tasks.
- **settings.gradle**: Gradle project settings.
- **README.md**: This documentation file.
- **launch.json**: VS Code debug configuration.

## 2. Database

- **database/finance.db**: (Legacy) SQLite database file. Not used in cloud hosting; replaced by PostgreSQL.
- **libs/sqlite-jdbc-3.36.0.3.jar**: (Legacy) SQLite JDBC driver. Not used in cloud hosting.

## 3. Resources

- **resources/icons/**: App icons and favicon.
- **resources/css/theme.css**: Custom CSS theme for UI.

## 4. Templates (UI)

Located in `src/main/resources/templates/`:
- **index.html**: Main dashboard landing page. Shows analytics, charts, and navigation for all roles.
- **user.html**: User dashboard. Allows users to add/view expenses and budgets. Integrates AI category suggestion.
- **admin.html**: Admin dashboard. Manage users and view user list.
- **advisor_user.html**: Advisor's view of a user's expenses.

## 5. Java Source Code

Located in `src/main/java/com/example/` and subfolders:

### Main Application
- **PersonalFinancePlatformApplication.java**: Spring Boot main entry point. Starts the web server.

### Controllers
- **controller/ApiController.java**: REST API endpoints for stats, user management, and AI features.
- **pfp/controller/HomeController.java**: Handles web page routing for Thymeleaf templates.

### Services
- **pfp/service/DatabaseService.java**: Main database logic using PostgreSQL. Handles CRUD for users, expenses, budgets, and demo data insertion.
- **service/StatsService.java**: (Legacy/compat) Analytics and stats logic (now migrated to PostgreSQL).
- **service/SimpleAICategoryService.java**: AI-powered category suggestion logic for expenses.
- **service/StatsServiceDisabled.java**: Placeholder/disabled service (not used).

### Models
- **Models/User.java**: User entity/model.
- **Models/Expense.java**: Expense entity/model.
- **Models/Budget.java**: Budget entity/model.

### Legacy
- **Database.java**: Old database logic for SQLite (not used in cloud hosting).
- **UserDashboard.java, AdminDashboard.java, AdvisorDashboard.java, Main.java**: Old UI logic (replaced by Thymeleaf templates).

---

## 6. Configuration

- **src/main/resources/application.properties**: Main Spring Boot and database configuration. Set for PostgreSQL and cloud hosting.

---

## 7. How the Code Works (Summary)

- **Spring Boot** starts the app and serves web pages and REST APIs.
- **Thymeleaf templates** render dynamic HTML for each user role.
- **DatabaseService** manages all data in PostgreSQL (users, expenses, budgets, demo data).
- **ApiController** provides REST endpoints for analytics, user management, and AI features.
- **SimpleAICategoryService** gives AI-based category suggestions for expenses.
- **StatsService** (legacy) provides analytics logic (now migrated to PostgreSQL).
- **Models** define the structure of users, expenses, and budgets.
- **application.properties** configures the database and server for local/public hosting.

---

## 8. Hosting

- For local: Use Gradle to run the app, connect to local PostgreSQL.
- For public: Deploy to Render/Railway, set environment variables for PostgreSQL.

---

For more details, see comments in each file or ask for a specific file explanation.

---

# (Legacy content below for reference)

...existing code...
