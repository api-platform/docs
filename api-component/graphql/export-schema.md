## Export the Schema in SDL

You may need to export your schema in SDL (Schema Definition Language) to import it in some tools.

The `api:graphql:export` command is provided to do so:

```bash
bin/console api:graphql:export -o path/to/your/volume/schema.graphql
```

Since the command prints the schema to the output if you don't use the `-o` option, you can also use this command:

```bash
bin/console api:graphql:export > path/in/host/schema.graphql
```
