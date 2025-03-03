### **Introduction**

Converting Oracle Pro*C code to a **Java Spring** application involves several key steps, since **Oracle Pro*C** and **Java Spring** differ significantly in their structure and how they handle database interactions. Here's a breakdown of what I would need to do:

### **Understand the Oracle Pro*C Code**
Oracle Pro*C allows C programs to embed SQL queries directly into the C code, often using `EXEC SQL` statements. These Pro*C programs interact with an Oracle database and handle database operations like SELECT, INSERT, UPDATE, and DELETE, as well as procedural logic like variable declaration.

Sample **Oracle Pro*C** code
```c
#include <sqlca.h>  // Oracle C API header

EXEC SQL BEGIN DECLARE SECTION;
    int emp_id;
    char emp_name[100];
EXEC SQL END DECLARE SECTION;

int main() {
    /* Oracle database connection setup */
    /* Execute SQL */
    EXEC SQL SELECT employee_name INTO :emp_name
             FROM employees WHERE employee_id = :emp_id;
    
    /* Output result */
    printf("Employee Name: %s\n", emp_name);
    
    return 0;
}
```

### **1. The key aspects of **Oracle Pro*C** language**
- **SQL queries** are embedded within the C code.
- **Variable bindings**: Variables (e.g., `emp_id`, `emp_name`) are bound to SQL statements using the colon (`:`) syntax.
- **Oracle-specific API**: Pro*C relies on Oracle's `sqlca.h` header for managing database connections and handling errors.

### **2. Key Differences Between Oracle Pro*C and Java Spring**
- **JDBC (Java Database Connectivity)** or **JPA (Java Persistence API)** to interact with the database.
- **Spring Data** and **Spring JDBC** to manage database operations and transactions.
- **Spring Boot** to set up and manage the application lifecycle and services.

---

### Convert **Oracle Pro*C** to Java Spring illustration

- `PlSqlParser.g4`: https://github.com/antlr/grammars-v4/blob/master/sql/plsql/PlSqlParser.g4
- `PlSqlLexer.g4`: https://github.com/antlr/grammars-v4/blob/master/sql/plsql/PlSqlLexer.g4
- 

I will now illustrate how to extend the ANTLR grammar files (PlSqlLexer.g4 and PlSqlParser.g4) to handle the syntax of Oracle ProC embedded within C code. This code involves embedded SQL statements such as EXEC SQL SELECT, EXEC SQL INSERT, EXEC SQL DECLARE, and EXEC SQL FETCH.

### Step 1. Modify `PlSqlLexer.g4` (Lexer)

The lexer needs to capture tokens that correspond to the SQL keywords embedded within the C code. The relevant tokens will include `EXEC SQL`, SQL keywords (like `SELECT`, `INSERT`, `FETCH`, etc.), and C-specific constructs like `#include` and variable declarations.

#### Updated `PlSqlLexer.g4`

```antlr
lexer grammar PlSqlLexer;

SQL_BLOCK: 'EXEC SQL' ~[\r\n]*; // Match 'EXEC SQL' keyword and embedded SQL block

C_INCLUDE: '#include' ~[\r\n]*; // Match #include preprocessor directive in C

IDENTIFIER: [a-zA-Z_][a-zA-Z_0-9]*; // Match C variable names, function names, and SQL identifiers

NUMERIC_LITERAL: [0-9]+(\.[0-9]+)?; // Match numeric literals (integers or floats)

STRING_LITERAL: '"' (~["\r\n])* '"'; // Match string literals

COMMENT: '--' ~[\r\n]* -> skip; // Skip single-line comments in SQL

MULTILINE_COMMENT: '/*' .*? '*/' -> skip; // Skip multi-line comments in SQL

WS: [ \t\r\n]+ -> skip; // Skip whitespace characters

// SQL keywords
SELECT: 'SELECT' ~[\r\n]*;
INSERT: 'INSERT' ~[\r\n]*;
FETCH: 'FETCH' ~[\r\n]*;
OPEN: 'OPEN' ~[\r\n]*;
CLOSE: 'CLOSE' ~[\r\n]*;
DECLARE: 'DECLARE' ~[\r\n]*;
COMMIT: 'COMMIT' ~[\r\n]*;
EXEC: 'EXEC' ~[\r\n]*; // Recognize 'EXEC' part of EXEC SQL

// C data types and other keywords
INT: 'int' ~[\r\n]*;
CHAR: 'char' ~[\r\n]*;
FLOAT: 'float' ~[\r\n]*;
VOID: 'void' ~[\r\n]*;
```

### Key Points:
1. **SQL_BLOCK**: Matches any `EXEC SQL` block, allowing for flexible SQL parsing.
2. **C_INCLUDE**: Matches `#include` directives in C.
3. **SQL Keywords**: Tokens for SQL keywords like `SELECT`, `INSERT`, `FETCH`, `COMMIT`, etc., are added.
4. **C Data Types**: `int`, `char`, `float`, and other data types from the C code are captured.
5. **Whitespace and Comments**: SQL and C comments are skipped using `COMMENT` and `MULTILINE_COMMENT`.

### Step 2. Modify `PlSqlParser.g4` (Parser)

The parser needs to parse both C code and embedded SQL. It will distinguish between regular C constructs and SQL statements embedded using `EXEC SQL`.

#### Updated `PlSqlParser.g4`

```antlr
parser grammar PlSqlParser;

options {
  tokenVocab=PlSqlLexer;
}

program: (include_directive | sql_statement | c_code)*;

include_directive: C_INCLUDE STRING_LITERAL; // Handle #include directives

sql_statement: 
    exec_sql_statement
  | select_statement
  | insert_statement
  | fetch_statement
  | open_statement
  | close_statement
  | declare_statement
  | commit_statement;

exec_sql_statement: 'EXEC SQL' ~[\r\n]*; // Match general EXEC SQL statement

select_statement: 'SELECT' ~[\r\n]*; // Match SELECT statement in embedded SQL

insert_statement: 'INSERT' ~[\r\n]*; // Match INSERT statement in embedded SQL

fetch_statement: 'FETCH' ~[\r\n]*; // Match FETCH statement in embedded SQL

open_statement: 'OPEN' ~[\r\n]*; // Match OPEN statement in embedded SQL

close_statement: 'CLOSE' ~[\r\n]*; // Match CLOSE statement in embedded SQL

declare_statement: 'DECLARE' ~[\r\n]*; // Match DECLARE section in embedded SQL

commit_statement: 'COMMIT' ~[\r\n]*; // Match COMMIT statement in embedded SQL

c_code: c_variable_declaration | function_definition | other_c_code;

c_variable_declaration: data_type IDENTIFIER (',' IDENTIFIER)* ';'; // Match C variable declarations

function_definition: data_type IDENTIFIER '(' (parameter (',' parameter)*)? ')' '{' (c_code)* '}'; // Match C function definitions

other_c_code: ~[a-zA-Z]*; // Match any other C code (excluding keywords and variables)

data_type: INT | CHAR | FLOAT | VOID; // Match C data types

parameter: data_type IDENTIFIER; // Match C function parameters
```

### Key Points:
1. **program**: The root rule that processes a sequence of either include directives, SQL statements, or C code.
2. **sql_statement**: Captures various types of SQL statements (like `SELECT`, `INSERT`, `FETCH`, etc.) embedded within the `EXEC SQL` block.
3. **c_code**: Defines a set of rules for parsing regular C code constructs, including variable declarations, function definitions, and other code.
4. **c_variable_declaration**: Matches C variable declarations that are typically used in Pro*C to declare variables that will hold SQL query results.
5. **function_definition**: Handles C function definitions, enabling the parser to recognize function prototypes and bodies.
6. **other_c_code**: Captures any remaining C code that isn't directly related to SQL or function declarations.

### Example of Parsing the Given Pro*C Code


```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sqlca.h>
#include <sqlcpr.h>

#define MAX_QUERY_LENGTH 1024
#define MAX_BIND_VARIABLES 10

typedef struct {
    int id;
    char name[50];
    float salary;
} Employee;

EXEC SQL INCLUDE sqlca;

void check_sql_error(const char* msg) {
    if (sqlca.sqlcode != 0) {
        printf("Error in %s: %s\n", msg, sqlca.sqlerrm.sqlmsg);
        exit(1);
    }
}

void fetch_employees() {
    EXEC SQL DECLARE emp_cursor CURSOR FOR emp_cursor_query;
    EXEC SQL OPEN emp_cursor;

    Employee emp;
    int count = 0;

    printf("Fetching employees...\n");

    while (1) {
        EXEC SQL FETCH emp_cursor INTO :emp.id, :emp.name, :emp.salary;

        if (sqlca.sqlcode == 1403) {
            break;
        }
        check_sql_error("fetching employee data");

        printf("Employee #%d: %s, Salary: %.2f\n", emp.id, emp.name, emp.salary);
        count++;
    }

    printf("Fetched %d employees.\n", count);

    EXEC SQL CLOSE emp_cursor;
}
```

When parsed with the modified ANTLR grammar, the code will produce an AST (abstract syntax tree) that recognizes:
1. **C includes**: The `#include` directives (like `#include <stdio.h>`) will be parsed by the `include_directive` rule.
2. **C declarations**: The `typedef struct` for the `Employee` structure will be captured by the `c_code` rule.
3. **Embedded SQL**: The `EXEC SQL DECLARE`, `EXEC SQL OPEN`, `EXEC SQL FETCH`, and `EXEC SQL CLOSE` will be captured by the respective `sql_statement` rules.

### Step 3: Converting to Java Spring

Once the Pro*C code is parsed into an AST, you can transform it into Java Spring code:
- **SQL Statements**: SQL statements like `SELECT`, `INSERT`, and `FETCH` will be converted into equivalent Java code using Spring's `JdbcTemplate` or JPA repository methods.
- **C Code**: The C functions and variables can be translated into Java methods and fields.
  - For example, `fetch_employees()` could be converted into a Spring method that fetches employee data from a database.
  - The `check_sql_error()` function can be replaced with exception handling in Java.

---


### Parsing and creating AST (abstract syntax tree)
To provide you with a demonstration of how an AST (Abstract Syntax Tree) and a token tree might look based on the Pro*C code and the extended ANTLR grammar, I'll outline both the **Token Tree** and **AST** based on the Pro*C code you provided, and I'll also offer the **JSON format** representation.

Let's first recap the Pro*C code you provided:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sqlca.h>
#include <sqlcpr.h>

#define MAX_QUERY_LENGTH 1024
#define MAX_BIND_VARIABLES 10

typedef struct {
    int id;
    char name[50];
    float salary;
} Employee;

EXEC SQL INCLUDE sqlca;

void check_sql_error(const char* msg) {
    if (sqlca.sqlcode != 0) {
        printf("Error in %s: %s\n", msg, sqlca.sqlerrm.sqlmsg);
        exit(1);
    }
}

void fetch_employees() {
    EXEC SQL DECLARE emp_cursor CURSOR FOR emp_cursor_query;
    EXEC SQL OPEN emp_cursor;

    Employee emp;
    int count = 0;

    printf("Fetching employees...\n");

    while (1) {
        EXEC SQL FETCH emp_cursor INTO :emp.id, :emp.name, :emp.salary;

        if (sqlca.sqlcode == 1403) {
            break;
        }
        check_sql_error("fetching employee data");

        printf("Employee #%d: %s, Salary: %.2f\n", emp.id, emp.name, emp.salary);
        count++;
    }

    printf("Fetched %d employees.\n", count);

    EXEC SQL CLOSE emp_cursor;
}
```

### 1. Token Tree Example:
In ANTLR, the lexer generates tokens that represent the components of the input code. For the Pro*C code provided, the token tree will consist of tokens like `C_INCLUDE`, `IDENTIFIER`, `EXEC`, `SQL`, `SELECT`, `DECLARE`, etc.

Let's break down the token structure.

#### Sample Token Tree for the Pro*C Code:

```text
[PROGRAM]
  ├── [INCLUDE_DIRECTIVE]
  │   ├── #include
  │   └── <stdio.h>
  ├── [INCLUDE_DIRECTIVE]
  │   ├── #include
  │   └── <stdlib.h>
  ├── [INCLUDE_DIRECTIVE]
  │   ├── #include
  │   └── <string.h>
  ├── [INCLUDE_DIRECTIVE]
  │   ├── #include
  │   └── <sqlca.h>
  ├── [INCLUDE_DIRECTIVE]
  │   ├── #include
  │   └── <sqlcpr.h>
  ├── [DEFINE]
  │   ├── #define
  │   ├── MAX_QUERY_LENGTH
  │   └── 1024
  ├── [DEFINE]
  │   ├── #define
  │   ├── MAX_BIND_VARIABLES
  │   └── 10
  ├── [STRUCT_DECLARATION]
  │   ├── typedef
  │   ├── struct
  │   ├── Employee
  │   ├── int id
  │   ├── char name[50]
  │   └── float salary
  ├── [EXEC_SQL_STATEMENT]
  │   ├── EXEC
  │   ├── SQL
  │   ├── INCLUDE
  │   └── sqlca
  ├── [FUNCTION_DECLARATION]
  │   ├── void
  │   ├── check_sql_error
  │   ├── (const char* msg)
  │   └── [FUNCTION_BODY]
  ├── [FUNCTION_DECLARATION]
  │   ├── void
  │   ├── fetch_employees
  │   ├── ()
  │   └── [FUNCTION_BODY]
  │       ├── EXEC SQL DECLARE
  │       ├── EXEC SQL OPEN
  │       ├── EXEC SQL FETCH
  │       ├── EXEC SQL CLOSE
```
- **Tokens** represent the low-level components of the Pro*C code (e.g., `#include`, `typedef`, `EXEC SQL`).

### 2. Abstract Syntax Tree (AST) Example:
The Abstract Syntax Tree represents the hierarchical structure of the program. Each node in the tree corresponds to a different type of statement or construct in the code.

#### Sample AST for the Pro*C Code:

```json
{
  "type": "Program",
  "children": [
    {
      "type": "IncludeDirective",
      "name": "#include <stdio.h>"
    },
    {
      "type": "IncludeDirective",
      "name": "#include <stdlib.h>"
    },
    {
      "type": "IncludeDirective",
      "name": "#include <string.h>"
    },
    {
      "type": "IncludeDirective",
      "name": "#include <sqlca.h>"
    },
    {
      "type": "IncludeDirective",
      "name": "#include <sqlcpr.h>"
    },
    {
      "type": "Define",
      "define": {
        "name": "MAX_QUERY_LENGTH",
        "value": 1024
      }
    },
    {
      "type": "Define",
      "define": {
        "name": "MAX_BIND_VARIABLES",
        "value": 10
      }
    },
    {
      "type": "StructDeclaration",
      "name": "Employee",
      "fields": [
        {"type": "int", "name": "id"},
        {"type": "char[50]", "name": "name"},
        {"type": "float", "name": "salary"}
      ]
    },
    {
      "type": "ExecSqlStatement",
      "sql": "INCLUDE sqlca"
    },
    {
      "type": "FunctionDeclaration",
      "name": "check_sql_error",
      "parameters": [
        {"type": "const char*", "name": "msg"}
      ],
      "body": [
        {
          "type": "IfStatement",
          "condition": "sqlca.sqlcode != 0",
          "body": [
            {
              "type": "PrintStatement",
              "message": "Error in %s: %s\n"
            },
            {
              "type": "ExitStatement"
            }
          ]
        }
      ]
    },
    {
      "type": "FunctionDeclaration",
      "name": "fetch_employees",
      "parameters": [],
      "body": [
        {
          "type": "ExecSqlStatement",
          "sql": "DECLARE emp_cursor CURSOR FOR emp_cursor_query"
        },
        {
          "type": "ExecSqlStatement",
          "sql": "OPEN emp_cursor"
        },
        {
          "type": "WhileLoop",
          "condition": "1",
          "body": [
            {
              "type": "ExecSqlStatement",
              "sql": "FETCH emp_cursor INTO :emp.id, :emp.name, :emp.salary"
            },
            {
              "type": "IfStatement",
              "condition": "sqlca.sqlcode == 1403",
              "body": [
                {
                  "type": "BreakStatement"
                }
              ]
            },
            {
              "type": "CheckSqlError",
              "message": "fetching employee data"
            },
            {
              "type": "PrintStatement",
              "message": "Employee #%d: %s, Salary: %.2f\n"
            }
          ]
        },
        {
          "type": "PrintStatement",
          "message": "Fetched %d employees.\n"
        },
        {
          "type": "ExecSqlStatement",
          "sql": "CLOSE emp_cursor"
        }
      ]
    }
  ]
}
```

- **AST** is a more structured representation of the code, showing the relationships between constructs (e.g., function declarations, SQL statements, structs).

### 3. JSON Representation of Tokens and AST for Pro*C:

The **tokens** and **AST** can be represented in JSON format to allow easy traversal or transformation. The tokens represent the lexer's output, and the AST represents the hierarchical structure of the parsed code.

#### Sample JSON Token Tree:

```json
{
  "tokens": [
    {"type": "C_INCLUDE", "value": "#include <stdio.h>"},
    {"type": "C_INCLUDE", "value": "#include <stdlib.h>"},
    {"type": "C_INCLUDE", "value": "#include <string.h>"},
    {"type": "C_INCLUDE", "value": "#include <sqlca.h>"},
    {"type": "C_INCLUDE", "value": "#include <sqlcpr.h>"},
    {"type": "DEFINE", "value": "#define MAX_QUERY_LENGTH 1024"},
    {"type": "DEFINE", "value": "#define MAX_BIND_VARIABLES 10"},
    {"type": "IDENTIFIER", "value": "typedef"},
    {"type": "IDENTIFIER", "value": "struct"},
    {"type": "IDENTIFIER", "value": "Employee"},
    {"type": "TYPE", "value": "int"},
    {"type": "IDENTIFIER", "value": "id"},
    {"type": "TYPE", "value": "char"},
    {"type": "IDENTIFIER", "value": "name"},
    {"type": "ARRAY_SIZE", "value": "50"},
    {"type": "TYPE", "value": "float"},
    {"type": "IDENTIFIER", "value": "salary"},
    {"type": "EXEC_SQL", "value": "EXEC SQL INCLUDE sqlca"},
    {"type": "FUNCTION_DECLARATION", "value": "check_sql_error"},
    {"type": "FUNCTION_DECLARATION", "value": "fetch_employees"}
  ]
}
```

- The **JSON format** provides an easy-to-parse representation that can be used in tools or further transformations.










































### Parsing and Converting to Java.

### Step 1: **Program to Parse and use the Visitor**

Program to use the visitor class after parsing the input code.


```csharp
using System;
using System.IO;
using Antlr4.Runtime;

namespace ProCParser
{
    class Program
    {
        static void Main(string[] args)
        {
            string inputCode = @"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sqlca.h>
#include <sqlcpr.h>

#define MAX_QUERY_LENGTH 1024
#define MAX_BIND_VARIABLES 10

typedef struct {
    int id;
    char name[50];
    float salary;
} Employee;

EXEC SQL INCLUDE sqlca;

void check_sql_error(const char* msg) {
    if (sqlca.sqlcode != 0) {
        printf('Error in %s: %s\n', msg, sqlca.sqlerrm.sqlmsg);
        exit(1);
    }
}

void fetch_employees() {
    EXEC SQL DECLARE emp_cursor CURSOR FOR emp_cursor_query;
    EXEC SQL OPEN emp_cursor;

    Employee emp;
    int count = 0;

    printf('Fetching employees...\n');

    while (1) {
        EXEC SQL FETCH emp_cursor INTO :emp.id, :emp.name, :emp.salary;

        if (sqlca.sqlcode == 1403) {
            break;
        }
        check_sql_error('fetching employee data');

        printf('Employee #%d: %s, Salary: %.2f\n', emp.id, emp.name, emp.salary);
        count++;
    }

    printf('Fetched %d employees.\n', count);

    EXEC SQL CLOSE emp_cursor;
}
";

            // Create an input stream from the Pro*C code
            AntlrInputStream inputStream = new AntlrInputStream(inputCode);

            // Create the lexer
            PlSqlLexer lexer = new PlSqlLexer(inputStream);

            // Create a token stream
            CommonTokenStream commonTokenStream = new CommonTokenStream(lexer);

            // Create the parser
            PlSqlParser parser = new PlSqlParser(commonTokenStream);

            // Start parsing the program (or root rule in your grammar)
            var context = parser.program();

            // Create an instance of the visitor
            var visitor = new ProCVisitor();

            // Visit the parse tree and print out different parts of the code
            visitor.Visit(context);

            Console.WriteLine("Parsing completed.");
        }
    }
}
```


### Step 2: **Design the Unified Visitor Class**

- In this step, we will create a unified visitor class that can process both **Pro*C constructs** (such as SQL statements and C code) and **arithmetic expressions**. The visitor will inherit from the `PlSqlBaseVisitor`, which is automatically generated by ANTLR, and we will override specific methods to handle different nodes in the parse tree. By combining the functionality of both the **ProCVisitor** and the **ArithmeticVisitor**, we will create a single "master" visitor that can manage multiple kinds of nodes and expressions in the parse tree.

- Instead of maintaining separate visitors for Pro*C constructs and arithmetic expressions, we integrate them into one comprehensive visitor. This approach allows the visitor to handle a wide range of constructs, including Pro*C SQL statements, C code, and arithmetic operations such as addition, subtraction, multiplication, division, and parenthesized expressions.

### The combined visitor will:
- Process Pro*C SQL statements (like `EXEC SQL`, `SELECT`, `INSERT`).
- Handle arithmetic expressions (like `+`, `-`, `*`, `/`) and nested expressions.
- Provide a clear, modular way to manage different types of language constructs in a single visitor class.


Here’s a unified visitor that can handle both:

### Code for Combined Visitor

```csharp
using Antlr4.Runtime.Misc;
using Antlr4.Runtime.Tree;
using System;

namespace ProCParser
{
    // The combined visitor inherits from both ProC and Arithmetic base visitors
    public class MasterVisitor : PlSqlBaseVisitor<object>
    {
        // Visit method for handling all types of expressions and SQL statements
        public override object Visit([NotNull] IParseTree tree)
        {
            // Switch based on the specific context type
            switch (tree)
            {
                // Handling SQL EXEC statements (e.g., EXEC SQL DECLARE, EXEC SQL SELECT, etc.)
                case PlSqlParser.Exec_sql_statementContext execContext:
                    return VisitExec_sql_statement(execContext);

                // Handling SQL SELECT statements
                case PlSqlParser.Select_statementContext selectContext:
                    return VisitSelect_statement(selectContext);

                // Handling SQL INSERT statements
                case PlSqlParser.Insert_statementContext insertContext:
                    return VisitInsert_statement(insertContext);

                // Handling C variable declarations
                case PlSqlParser.C_variable_declarationContext varDeclContext:
                    return VisitC_variable_declaration(varDeclContext);

                // Handling C function definitions
                case PlSqlParser.Function_definitionContext funcDefContext:
                    return VisitFunction_definition(funcDefContext);

                // Handling other types of C code
                case PlSqlParser.Other_c_codeContext otherCodeContext:
                    return VisitOther_c_code(otherCodeContext);

                // Handling arithmetic operations (addition, subtraction, etc.)
                case ArithmeticParser.AddSubContext addSubContext:
                    return VisitAddSub(addSubContext);

                case ArithmeticParser.MulDivContext mulDivContext:
                    return VisitMulDiv(mulDivContext);

                case ArithmeticParser.ParensContext parensContext:
                    return VisitParens(parensContext);

                case ArithmeticParser.NumberContext numberContext:
                    return VisitNumber(numberContext);

                default:
                    Console.WriteLine("Unknown context: " + tree.GetType().Name);
                    return base.Visit(tree);
            }
        }

        // Handle SQL EXEC statements (e.g., EXEC SQL DECLARE, EXEC SQL SELECT, etc.)
        public object VisitExec_sql_statement(PlSqlParser.Exec_sql_statementContext context)
        {
            Console.WriteLine("Visiting EXEC SQL statement: " + context.GetText());
            return base.VisitExec_sql_statement(context);
        }

        // Handle SQL SELECT statements
        public object VisitSelect_statement(PlSqlParser.Select_statementContext context)
        {
            Console.WriteLine("Visiting SELECT statement: " + context.GetText());
            return base.VisitSelect_statement(context);
        }

        // Handle SQL INSERT statements
        public object VisitInsert_statement(PlSqlParser.Insert_statementContext context)
        {
            Console.WriteLine("Visiting INSERT statement: " + context.GetText());
            return base.VisitInsert_statement(context);
        }

        // Handle C variable declarations
        public object VisitC_variable_declaration(PlSqlParser.C_variable_declarationContext context)
        {
            Console.WriteLine("Visiting C variable declaration: " + context.GetText());
            return base.VisitC_variable_declaration(context);
        }

        // Handle C function definitions
        public object VisitFunction_definition(PlSqlParser.Function_definitionContext context)
        {
            Console.WriteLine("Visiting function definition: " + context.GetText());
            return base.VisitFunction_definition(context);
        }

        // Handle other types of C code
        public object VisitOther_c_code(PlSqlParser.Other_c_codeContext context)
        {
            Console.WriteLine("Visiting other C code: " + context.GetText());
            return base.VisitOther_c_code(context);
        }

        // Handling arithmetic expressions (additions and subtractions)
        public int VisitAddSub(ArithmeticParser.AddSubContext context)
        {
            int left = VisitArithmeticExpression(context.expr(0));  // Visit the left operand
            int right = VisitArithmeticExpression(context.expr(1)); // Visit the right operand
            string operatorSymbol = context.op.Text; // "+" or "-"
            
            if (operatorSymbol == "+")
                return left + right;
            else
                return left - right;
        }

        // Handling multiplication and division
        public int VisitMulDiv(ArithmeticParser.MulDivContext context)
        {
            int left = VisitArithmeticExpression(context.expr(0));  // Visit the left operand
            int right = VisitArithmeticExpression(context.expr(1)); // Visit the right operand
            string operatorSymbol = context.op.Text; // "*" or "/"
            
            if (operatorSymbol == "*")
                return left * right;
            else
                return left / right;
        }

        // Handling parenthesized expressions
        public int VisitParens(ArithmeticParser.ParensContext context)
        {
            return VisitArithmeticExpression(context.expr());  // Visit the nested expression inside parentheses
        }

        // Handling numbers (literal values)
        public int VisitNumber(ArithmeticParser.NumberContext context)
        {
            return int.Parse(context.GetText());  // Convert the text to an integer
        }

        // Helper method to visit arithmetic expressions (to reduce duplication)
        private int VisitArithmeticExpression(ArithmeticParser.ExprContext exprContext)
        {
            // This handles the recursion in the case of nested expressions
            return (int)Visit(exprContext);  // Recursively visit the sub-expression
        }
    }
}
```

### Explanation of the Combined Visitor:

#### 1. **Main Visit Method:**
   - The `Visit` method is overridden to handle different types of contexts based on the parse tree's node type. It can handle both the Pro*C constructs (like `EXEC SQL`, `INSERT`, etc.) and the arithmetic expressions (`AddSub`, `MulDiv`, `Parens`, etc.).
   - It uses a `switch` statement to differentiate between different contexts.
   - Each case calls a specific visitor method for SQL statements (like `VisitExec_sql_statement`) or for arithmetic expressions (like `VisitAddSub`).

#### 2. **SQL Statement Handlers:**
   - These methods (`VisitExec_sql_statement`, `VisitSelect_statement`, `VisitInsert_statement`, etc.) are responsible for processing SQL-specific code like `EXEC SQL DECLARE` or `SELECT` statements.
   - Each SQL statement method simply prints out the content and calls `base.Visit` for any additional handling.

#### 3. **Arithmetic Expression Handlers:**
   - The arithmetic expression methods (`VisitAddSub`, `VisitMulDiv`, `VisitParens`, etc.) handle the arithmetic operations like addition, subtraction, multiplication, division, and nested expressions inside parentheses.
   - For operations like addition and subtraction (`AddSub`), we recursively visit the left and right operands and then perform the operation based on the operator.
   - For multiplication and division (`MulDiv`), the same logic applies.
   - For parenthesized expressions (`Parens`), we visit the inner expression and return its result.

### 4. **Handling Nested Expressions:**
   - The `VisitArithmeticExpression` method allows for recursive evaluation of nested arithmetic expressions. If an expression contains other expressions inside parentheses, the method will handle it in a recursive manner, ensuring that deeply nested expressions are evaluated correctly.
