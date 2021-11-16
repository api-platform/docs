# Custom Generator

You will probably want to extend or, at least, take a look at [BaseGenerator.js](https://github.com/api-platform/client-generator/blob/main/src/generators/BaseGenerator.js), since the library expects some methods to be available, as well as one of the included generator to make your own.

## Usage

```shell
generate-api-platform-client -g "$(pwd)/path/to/custom/generator.js" -t "$(pwd)/path/to/templates"
```

The `-g` argument can point to any resolvable node module which means it can be a package dependency of the current project as well as any js file.

## Example

Let's create a basic react generator with Create form as an example:

### Generator

```js
// ./Generator.js
import BaseGenerator from "@api-platform/client-generator/lib/generators/BaseGenerator";

export default class extends BaseGenerator {
    constructor(params) {
        super(params);

        this.registerTemplates("", [
            "utils/dataAccess.js",
            "components/foo/Create.js",
            "routes/foo.js",
        ]);
    }

    help(resource) {
        const titleLc = resource.title.toLowerCase();

        console.log(
            'Code for the "%s" resource type has been generated!',
            resource.title
        );
        console.log(`
//import routes
import ${titleLc}Routes from './routes/${titleLc}';

// Add routes to <Switch>
{ ${titleLc}Routes }
`);
    }

    generate(api, resource, dir) {
        const lc = resource.title.toLowerCase();
        const titleUcFirst =
            resource.title.charAt(0).toUpperCase() + resource.title.slice(1);

        const context = {
            title: resource.title,
            name: resource.name,
            lc,
            uc: resource.title.toUpperCase(),
            fields: resource.readableFields,
            formFields: this.buildFields(resource.writableFields),
            hydraPrefix: this.hydraPrefix,
            titleUcFirst,
        };

        // Create directories
        // These directories may already exist
        [`${dir}/utils`, `${dir}/config`, `${dir}/routes`].forEach((dir) =>
            this.createDir(dir, false)
        );

        [`${dir}/components/${lc}`].forEach((dir) => this.createDir(dir));

        ["components/%s/Create.js", "routes/%s.js"].forEach((pattern) =>
            this.createFileFromPattern(pattern, dir, lc, context)
        );

        // utils
        this.createFile(
            "utils/dataAccess.js",
            `${dir}/utils/dataAccess.js`,
            context,
            false
        );

        this.createEntrypoint(api.entrypoint, `${dir}/config/entrypoint.js`);
    }
}
```

### `Create` component

```js
// template/components/Create.js
import React from 'react';
import { Redirect } from 'react-router-dom';
import fetch from '../utils/dataAccess';

export default function Create() {
  const [isLoading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [created, setCreated] = useState(null);

  const create = useCallback(async (e) => {
    setLoading(true)
    try {
      const values = Array.from(e.target.elements).reduce((vals, e) => {
        vals[e.id] = e.value;
        return vals
      }, {})
      const response = await fetch('{{{name}}}', { method: 'POST', body: JSON.stringify(values) });
      const retrieved = await response.json();
      setCreated(retrieved);
    } catch (err) {
      setError(err);
    } finally {
      setLoading(false);
    }
  }, [setLoading, setError])

  if (created) {
    return <Redirect to={`edit/${encodeURIComponent(created['@id'])}`} />;
  }

  return (
    <div>
      <h1>New {{{title}}}</h1>

      {isLoading && (
        <div className="alert alert-info" role="status">
          Loading...
        </div>
      )}
      {error && (
        <div className="alert alert-danger" role="alert">
          <span className="fa fa-exclamation-triangle" aria-hidden="true" />{' '}
          {error}
        </div>
      )}

      <form onSubmit={create}>
{{#each formFields}}
        <div className={`form-group`}>
          <label
            htmlFor={`{{{lc}}}_{{{name}}}`}
            className="form-control-label"
          >
            {data.input.name}
          </label>
          <input
            name="{{{name}}}"
            type="{{{type}}}"{{#if step}}
            step="{{{step}}}"{{/if}}
            placeholder="{{{description}}}"{{#if required}}
            required={true}{{/if}}
            id={`{{{lc}}}_{{{name}}}`}
          />
        </div>

        <button type="submit" className="btn btn-success">
          Submit
        </button>
      </form>
    </div>
  );
}
```

### Utilities

```js
// template/entrypoint.js
export const ENTRYPOINT = "{{{entrypoint}}}";
```

```js
// template/routes/foo.js
import React from "react";
import { Route } from "react-router-dom";
import { Create } from "../components/{{{lc}}}/";

export default [
    <Route path="/{{{name}}}/create" component={Create} exact key="create" />,
];
```

```js
// template/utils/dataAccess.js
import { ENTRYPOINT } from "../config/entrypoint";
import { SubmissionError } from "redux-form";
import get from "lodash/get";
import has from "lodash/has";
import mapValues from "lodash/mapValues";

const MIME_TYPE = "application/ld+json";

export function fetch(id, options = {}) {
    if ("undefined" === typeof options.headers) options.headers = new Headers();
    if (null === options.headers.get("Accept"))
        options.headers.set("Accept", MIME_TYPE);

    if (
        "undefined" !== options.body &&
        !(options.body instanceof FormData) &&
        null === options.headers.get("Content-Type")
    )
        options.headers.set("Content-Type", MIME_TYPE);

    return global.fetch(new URL(id, ENTRYPOINT), options).then((response) => {
        if (response.ok) return response;

        return response.json().then(
            (json) => {
                const error =
                    json["hydra:description"] ||
                    json["hydra:title"] ||
                    "An error occurred.";
                if (!json.violations) throw Error(error);

                let errors = { _error: error };
                json.violations.forEach((violation) =>
                    errors[violation.propertyPath]
                        ? (errors[violation.propertyPath] +=
                              "\n" + errors[violation.propertyPath])
                        : (errors[violation.propertyPath] = violation.message)
                );

                throw new SubmissionError(errors);
            },
            () => {
                throw new Error(response.statusText || "An error occurred.");
            }
        );
    });
}

export function mercureSubscribe(url, topics) {
    topics.forEach((topic) =>
        url.searchParams.append("topic", new URL(topic, ENTRYPOINT))
    );

    return new EventSource(url.toString());
}

export function normalize(data) {
    if (has(data, "hydra:member")) {
        // Normalize items in collections
        data["hydra:member"] = data["hydra:member"].map((item) =>
            normalize(item)
        );

        return data;
    }

    // Flatten nested documents
    return mapValues(data, (value) =>
        Array.isArray(value)
            ? value.map((v) => normalize(v))
            : value instanceof Object
            ? normalize(value)
            : get(value, "@id", value)
    );
}

export function extractHubURL(response) {
    const linkHeader = response.headers.get("Link");
    if (!linkHeader) return null;

    const matches = linkHeader.match(
        /<([^>]+)>;\s+rel=(?:mercure|"[^"]*mercure[^"]*")/
    );

    return matches && matches[1] ? new URL(matches[1], ENTRYPOINT) : null;
}
```

Then we can use our generator:

```shell
generate-api-platform-client https://demo.api-platform.com out/ -g "$(pwd)/Generator.js" -t "$(pwd)/template"
```
