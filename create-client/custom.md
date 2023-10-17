# Custom Generator

Create Client provides support for many of the popular JS frameworks, but you may be using another framework or language and may need a solution adapted to your specific needs. For this scenario, you can write your own generator and pass it to the CLI using a path as the `-g` argument.

You will probably want to extend or, at least, take a look at [BaseGenerator.js](https://github.com/api-platform/create-client/blob/main/src/generators/BaseGenerator.js), since the library expects some methods to be available, as well as one of the [included generators](https://github.com/api-platform/create-client/blob/main/src/generators/BaseGenerator.js) to make your own.

## Usage

```shell
npm init @api-platform/client -- --generator "$(pwd)/path/to/custom/generator.js" -t "$(pwd)/path/to/templates"
```

The `-g` argument can point to any resolvable node module which means it can be a package dependency of the current project as well as any js file.

## Example

Create Client makes use of the [Handlebars](https://handlebarsjs.com/) template engine. You can use any programming language or file type. Your generator can also pass data to your templates in any shape you want.

In this example, we'll create a simple [Rust](https://www.rust-lang.org) file defining a new `struct` and creating some instances of this `struct`.

### Generator

```js
// ./Generator.js
import BaseGenerator from "@api-platform/create-client/lib/generators/BaseGenerator";

export default class extends BaseGenerator {
    constructor(params) {
        super(params);

        this.registerTemplates("", ["main.rs"]);
    }

    help() {}

    generate(api, resource, dir) {
        const context = {
            type: "Tilia",
            structure: [
                { name: "name", type: "String" },
                { name: "min_size", type: "u8" },
                { name: "max_size", type: "u8" },
            ],
            list: [
                {
                    name: "Tilia cordata",
                    minSize: 50,
                    maxSize: 80,
                },
                {
                    name: "Tilia platyphyllos",
                    minSize: 50,
                    maxSize: 70,
                },
                {
                    name: "Tilia tomentosa",
                    minSize: 50,
                    maxSize: 70,
                },
                {
                    name: "Tilia intermedia",
                    minSize: 50,
                    maxSize: 165,
                },
            ],
        };

        this.createDir(dir);

        this.createFile("main.rs", `${dir}/main.rs`, context, false);
    }
}
```

### Template

```rs
// template/main.rs
struct {{{type}}} {
  {{#each structure}}
  {{{name}}}: {{{type}}}
  {{/each}}
}

fn main() {
  let tilias = [
  {{#each list}}
    Tilia { name: "{{{name}}}", min_size: {{{minSize}}}, max_size: {{{maxSize}}}, },
  {{/each}}
  ];
}
```

Then we can use our generator:

```shell
npm init @api-platform/client https://demo.api-platform.com out/ -g "$(pwd)/Generator.js" -t "$(pwd)/template"
```

which will produces:

```ts
struct Tilia {
  name: String
  min_size: u8
  max_size: u8
}

fn main() {
  let tilias = [
    Tilia { name: "Tilia cordata", min_size: 50, max_size: 80, },
    Tilia { name: "Tilia platyphyllos", min_size: 50, max_size: 70, },
    Tilia { name: "Tilia tomentosa", min_size: 50, max_size: 70, },
    Tilia { name: "Tilia intermedia", min_size: 50, max_size: 165, },
  ];
}
```
