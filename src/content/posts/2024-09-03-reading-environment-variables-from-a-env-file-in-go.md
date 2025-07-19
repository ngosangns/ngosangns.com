---
published: 2024-09-03
title: Reading Environment Variables from a .env File in Go
tags: ["Go", "Golang", "Environment Variables", "Reflection", "Configuration", "Programming", "Backend", "Tutorial"]
category: "Backend Development"
---
In this post, we’ll explore a Go function that reads environment variables from a `.env` file and populates a struct with these values. This approach leverages Go’s reflection capabilities to dynamically set struct fields based on the environment variables.

Here’s the complete code for the function:

```go
package wire

import (
    _ "embed"
    "fmt"
    "reflect"
    "strings"
)

//go:embed .env
var envStr string

type Env struct {
    ENV        string
    DEV_EMAIL  string
    JWT_SECRET string
}

func ReadEnvFromEnvFile() *Env {
    _instance := &Env{}
    v := reflect.ValueOf(_instance).Elem()

    // split envStr into lines
    lines := strings.Split(envStr, "\n")

    // iterate over lines
    for _, line := range lines {
        // Skip empty lines and lines starting with #
        if strings.TrimSpace(line) == "" || strings.HasPrefix(line, "#") {
            continue
        }

        // Remove comments
        line = strings.SplitN(line, "#", 2)[0]

        // Split the line into key and value
        parts := strings.SplitN(line, "=", 2)
        if len(parts) != 2 {
            fmt.Println("Invalid line in .env file:", line)
            continue
        }

        key := strings.TrimSpace(parts[0])
        value := strings.TrimSpace(parts[1])

        // Use reflection to check if the key (field) exists in the struct
        f := v.FieldByName(key)

        if f.IsValid() {
            // If the field exists and is settable, update its value
            f.SetString(value)
        }
    }

    return _instance
}
```

#### Explanation

1. **Embedding the** `.env` File:

    ```go
    //go:embed .env
    var envStr string
    ```

    This line uses Go’s `embed` package to embed the `.env` file into the binary. The contents of the `.env` file are stored in the `envStr` variable.

2. **Defining the Struct**:

    ```go
    type Env struct {
        ENV        string
        DEV_EMAIL  string
        JWT_SECRET string
    }
    ```

    The `Env` struct represents the environment variables that we expect to find in the `.env` file.

3. **Reading and Parsing the** `.env` File:

    ```go
    lines := strings.Split(envStr, "\n")
    for _, line := range lines {
        if strings.TrimSpace(line) == "" || strings.HasPrefix(line, "#") {
            continue
        }
        line = strings.SplitN(line, "#", 2)[0]
        parts := strings.SplitN(line, "=", 2)
        if len(parts) != 2 {
            fmt.Println("Invalid line in .env file:", line)
            continue
        }
        key := strings.TrimSpace(parts[0])
        value := strings.TrimSpace(parts[1])
        f := v.FieldByName(key)
        if f.IsValid() {
            f.SetString(value)
        }
    }
    ```

    * **Skipping Invalid Lines**: Empty lines and comments are ignored.

    * **Removing Comments**: Anything after a `#` is considered a comment and removed.

    * **Splitting Key-Value Pairs**: Each line is split into key and value.

    * **Setting Struct Fields**: Using reflection, the struct fields are updated based on the key-value pairs.

---

This approach provides a flexible way to manage environment variables by leveraging reflection in Go. It allows for dynamic updates to struct fields based on environment configuration, making it easier to manage settings without hardcoding values into your application. If you have any specific requirements or questions, feel free to ask!
