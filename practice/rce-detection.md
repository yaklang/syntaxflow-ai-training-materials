# Practice: RCE Detection with ProcessBuilder

## Objective

Detect potential Remote Code Execution (RCE) vulnerabilities where user-controlled input flows into `ProcessBuilder` or `Runtime.exec` without proper sanitization.

## Rule: ProcessBuilder RCE

```syntaxflow
desc(title: "rce")

ProcessBuilder(* as $cmd) as $builder
$builder.start() as $execBuilder

check $execBuilder then "fine" else "rce 2 SyntaxFlow error"
alert $execBuilder for {
    message: "发现潜在的远程代码执行漏洞，未正确处理 ProcessBuilder 的参数。",
    risk: rce,
    level: high,
}
```

## Breakdown

- **`ProcessBuilder(* as $cmd)`**: Captures all constructor arguments into `$cmd` (the command and its arguments)
- **`$builder.start()`**: Tracks the `start()` call that actually executes the process
- **`check`**: Validates that the sink is reachable
- **`alert`**: Reports the vulnerability with metadata (message, risk, level)

## Alternative: Runtime.exec

For `Runtime.exec` style commands:

```syntaxflow
desc(title: "runtime-exec-rce")

Runtime.getRuntime().exec(* as $cmd)
$cmd #-> * as $source

check $source then "potential rce" else "fine"
alert $source for { message: "exec with user input", risk: rce }
```

## Key Concepts

- Use `* as $var` to capture arguments without naming them
- Chain `-->` for bottom-use (sink-to-source) or `#->` for top-def (source-to-definition)
- Combine with `<dataflow>` if you need to filter paths that pass through sanitizers
