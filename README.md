# -Rule-Engine-with-AST
The Rule Engine with AST is a 3-tier web application designed to evaluate user eligibility based on dynamic conditional rules. These rules are represented using an Abstract Syntax Tree (AST), allowing for flexibility in rule creation, combination, and modification.
Backend:Java,Springboot,Hibernate (Thymleaf)
Frontend:Html,Css,Javascript
Database:Mysql

1. Project Structure
   RuleEngine/
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/ruleengine/ast/
│   │   │       ├── controller/
│   │   │       │   └── RuleEngineController.java
│   │   │       ├── model/
│   │   │       │   └── Rule.java
│   │   │       │   └── Node.java
│   │   │       ├── service/
│   │   │       │   └── RuleEngineService.java
│   │   │       └── repository/
│   │   │           └── RuleRepository.java
│   │   ├── resources/
│   │   │   ├── templates/
│   │   │   │   └── index.html
│   │   │   └── application.properties
├── pom.xml
└── README.md

2.1 pom.xml (Dependencies)
<project ...>
    <dependencies>
        <!-- Spring Boot Dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
    </dependencies>
</project>

2.2 application.properties (MySQL Config)
spring.datasource.url=jdbc:mysql://localhost:3306/ruleengine
spring.datasource.username=root
spring.datasource.password=Naveenk
spring.jpa.hibernate.ddl-auto=update
spring.thymeleaf.cache=false

2.3 Node.java (AST Node Model)
package com.ruleengine.ast.model;

public class Node {
    private String type; // "operator" for AND/OR, "operand" for conditions
    private Node left;
    private Node right;
    private String value; // e.g. "age > 30", "salary > 50000"
    
    // Constructors, Getters, Setters
    public Node(String type, String value) {
        this.type = type;
        this.value = value;
    }
    public Node(String type, Node left, Node right) {
        this.type = type;
        this.left = left;
        this.right = right;
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }

    public Node getLeft() {
        return left;
    }

    public void setLeft(Node left) {
        this.left = left;
    }

    public Node getRight() {
        return right;
    }

    public void setRight(Node right) {
        this.right = right;
    }

    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }
}
2.4 Rule.java (Rule Entity for DB)
package com.ruleengine.ast.model;

import javax.persistence.*;

@Entity
public class Rule {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String ruleString;

    public Rule() {}

    public Rule(String ruleString) {
        this.ruleString = ruleString;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getRuleString() {
        return ruleString;
    }

    public void setRuleString(String ruleString) {
        this.ruleString = ruleString;
    }
}

2.5 RuleRepository.java (Repository for Rule)
package com.ruleengine.ast.repository;

import com.ruleengine.ast.model.Rule;
import org.springframework.data.jpa.repository.JpaRepository;

public interface RuleRepository extends JpaRepository<Rule, Long> {
}

2.6 RuleEngineService.java (Service for Rule Engine Logic)

package com.ruleengine.ast.service;

import com.ruleengine.ast.model.Node;
import com.ruleengine.ast.model.Rule;
import com.ruleengine.ast.repository.RuleRepository;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class RuleEngineService {

    private final RuleRepository ruleRepository;

    public RuleEngineService(RuleRepository ruleRepository) {
        this.ruleRepository = ruleRepository;
    }

    public Node createRule(String ruleString) {
        // Simple parser logic for demonstration
        // In reality, you'd need to tokenize the ruleString and build the AST
        String[] tokens = ruleString.split(" ");
        Node left = new Node("operand", tokens[0]);
        Node right = new Node("operand", tokens[2]);
        return new Node("operator", left, right);
    }

    public Node combineRules(Node rule1, Node rule2) {
        // Combine two rules using AND (for example)
        return new Node("operator", rule1, rule2);
    }

    public boolean evaluateRule(Node ast, Map<String, Object> data) {
        if ("operator".equals(ast.getType())) {
            boolean leftEval = evaluateRule(ast.getLeft(), data);
            boolean rightEval = evaluateRule(ast.getRight(), data);
            return leftEval && rightEval;
        } else {
            // Operand evaluation logic (e.g., "age > 30")
            return evaluateCondition(ast.getValue(), data);
        }
    }

    private boolean evaluateCondition(String condition, Map<String, Object> data) {
        // Simple condition evaluation, e.g., "age > 30"
        // Parse condition, extract field and compare
        String[] parts = condition.split(" ");
        String field = parts[0];
        String operator = parts[1];
        int value = Integer.parseInt(parts[2]);

        int fieldValue = (int) data.get(field);
        switch (operator) {
            case ">":
                return fieldValue > value;
            case "<":
                return fieldValue < value;
            default:
                return false;
        }
    }

    public Rule saveRule(String ruleString) {
        Rule rule = new Rule(ruleString);
        return ruleRepository.save(rule);
    }
}

2.7 RuleEngineController.java (Controller)
package com.ruleengine.ast.controller;

import com.ruleengine.ast.model.Node;
import com.ruleengine.ast.service.RuleEngineService;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

@Controller
public class RuleEngineController {

    private final RuleEngineService ruleEngineService;

    public RuleEngineController(RuleEngineService ruleEngineService) {
        this.ruleEngineService = ruleEngineService;
    }

    @GetMapping("/")
    public String index(Model model) {
        model.addAttribute("ruleString", "");
        return "index";
    }

    @PostMapping("/createRule")
    public String createRule(@RequestParam String ruleString, Model model) {
        Node ast = ruleEngineService.createRule(ruleString);
        model.addAttribute("ast", ast);
        return "result";
    }

    @PostMapping("/evaluateRule")
    public String evaluateRule(@RequestParam Map<String, String> params, Model model) {
        Map<String, Object> data = new HashMap<>();
        data.put("age", Integer.parseInt(params.get("age")));
        data.put("department", params.get("department"));
        data.put("salary", Integer.parseInt(params.get("salary")));
        data.put("experience", Integer.parseInt(params.get("experience")));

        // For simplicity, let's assume we are evaluating the first saved rule.
        Node ast = ruleEngineService.createRule(params.get("ruleString"));
        boolean result = ruleEngineService.evaluateRule(ast, data);
        model.addAttribute("result", result);
        return "result";
    }
}

2.8 index.html (Thymeleaf Template)
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Rule Engine</title>
</head>
<body>
<h1>Rule Engine with AST</h1>
<form action="/createRule" method="post">
    <label for="ruleString">Enter Rule String:</label>
    <input type="text" id="ruleString" name="ruleString">
    <button type="submit">Create Rule</button>
</form>
<br>
<form action="/evaluateRule" method="post">
    <label>Age:</label>
    <input type="number" name="age">
    <label>Department:</label>
    <input type="text" name="department">
    <label>Salary:</label>
    <input type="number" name="salary">
    <label>Experience:</label>
    <input type="number" name="experience">
    <input type="hidden" name="ruleString" value="${ruleString}">
    <button type="submit">Evaluate Rule</button>
</form>
</body>
</html>




