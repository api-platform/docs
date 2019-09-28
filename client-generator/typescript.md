# Typescript Interfaces

The TypeScript Generator allows you to create [TypeScript interfaces](https://www.typescriptlang.org/docs/handbook/interfaces.html) that you can embed in any TypeScript-enabled project (React, Vue.js, Angular..)

To do so, run the client generator:

    $ npx @api-platform/client-generator --generator typescript https://demo.api-platform.com src/ --resource foo
    # Replace the URL by the entrypoint of your Hydra-enabled API
    # "src/" represents where the interfaces will be generated
    # Omit the resource flag to generate files for all resource types exposed by the API

This command parses the Hydra documentation and creates one `.ts` file for each API Resource you have defined in your application, in the `interfaces` subfolder.

NOTE: If you are not sure what the entrypoint is, see [Troubleshooting](troubleshooting.md)

## Example

Assuming you have 2 resources in your application, `Foo` and `Bar`, when you run

    $ npx @api-platform/client-generator --generator typescript https://demo.api-platform.com src/

you will obtain 2 `.ts` files arranged as following:

* src/
  * interfaces/
    * foo.ts
    * bar.ts
