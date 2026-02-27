# Practice: XXE (XML External Entity) Detection

## Objective

Detect XML parsing that may be vulnerable to XXE when user-controlled input is passed to parsers like `DocumentBuilder.parse`, `SAXParser.parse`, or `XMLReader` without disabling external entities.

## Rule: DocumentBuilder.parse XXE

```syntaxflow
desc(title: "xxe-documentbuilder")

documentBuilder.parse(* as $input)
$input #-> * as $source

check $source then "potential xxe" else "fine"
alert $source for {
    message: "XML parse with external entity risk",
    risk: xxe,
    level: high,
}
```

## Breakdown

- **`documentBuilder.parse(* as $input)`**: Matches any `parse` call on a `documentBuilder`-like variable; `* as $input` captures the input (URL, InputStream, etc.)
- **`$input #-> * as $source`**: Top-def chain from the parse argument back to its definition
- **`check` / `alert`**: Standard reporting

## Alternative: SAXParser

```syntaxflow
desc(title: "xxe-saxparser")

saxParser.parse(* as $input, *)
$input #-> * as $source
alert $source for { risk: xxe }
```

## Key Concepts

- XXE often involves `DocumentBuilder`, `SAXParser`, `XMLReader`, `TransformerFactory`
- Capture the first argument of `parse(...)` as the user-controlled XML source
- Use `#->` to trace back to where the input originates (e.g., HTTP request, file)
