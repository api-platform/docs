# TypeScript Interfaces

The TypeScript Generator allows you to create [TypeScript interfaces](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#interfaces)
that you can embed in any TypeScript-enabled project (React, Vue.js, Angular..).

To do so, run the generator:

```console
npm init @api-platform/client https://demo.api-platform.com src/ -- --generator typescript --resource foo
# Replace the URL with the entrypoint of your Hydra-enabled API.
```

`src/` is where the interfaces will be generated.

Omit the resource flag to generate files for all resource types exposed by the API.
You can also use an OpenAPI documentation with `-f openapi3`.

This command parses the Hydra documentation and creates one `.ts` file for each API Resource you have defined in your application, in the `interfaces` subfolder.

**Note:** If you are not sure what the entrypoint is, see [Troubleshooting](troubleshooting.md).

## Example

Assuming you have 2 resources in your application, `Foo` and `Bar`, when you run:

```console
npm init @api-platform/client https://demo.api-platform.com src/ -- --generator typescript
```

You will obtain 2 `.ts` files arranged as following:

- src/
  - interfaces/
    - foo.ts
    - bar.ts
