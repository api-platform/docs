# Symfony maker commands

API Platform comes with a set of maker commands to help you create API Resources and other related
classes.

## Available commands

### Create an entity that is an API Resource

You can use Symfony
[MakerBundle](https://symfonycasts.com/screencast/symfony-fundamentals/maker-command?cid=apip) to
generate a Doctrine entity that is also a resource thanks to the `--api-resource` option:

```bash
bin/console make:entity --api-resource
```

### Create an API Filter

You can create an API Filter class using the following command:

```bash
bin/console make:filter <type> <name>
```

Where `<type>` is the filter type and `<name>` is the name of the filter class. Supported types are
`orm` and `odm`

> [!NOTE] Elasticsearch filters are not yet supported

### Create a State Provider

You can create a State Provider class using the following command:

```bash
bin/console make:state-provider <name>
```

### Create a State Processor

You can create a State Processor class using the following command:

```bash
bin/console make:state-processor <name>
```

## Configuration

You can disable the maker commands by setting the following configuration in your
`config/packages/api_platform.yaml` file:

```yaml
api_platform:
    maker: false
```

By default, the maker commands are enabled if the maker bundle is detected.

### Namespace configuration

The makers creates all classes in the configured maker bundle root_namespace (default `App`).
Filters are created in `App\\Filter` State Providers are created in `App\\State` State Processors
are created in `App\\State`

Should you customize the base namespace for all API Platform generated classes you can so in 2 ways:

- Bundle configuration
- Console Command Option

#### Bundle configuration

To change the default namespace prefix (relative to the maker.root_namespace), you can set the
following configuration in your `config/packages/api_platform.yaml` file:

```yaml
api_platform:
    maker:
        namespace_prefix: "Api"
```

#### Console Command Option

You can override the default namespace prefix by using the `--namespace-prefix` option when running
the maker commands:

```bash
bin/console make:filter orm MyCustomFilter --namespace-prefix Api\\Filter
bin/console make:state-provider MyProcessor --namespace-prefix Api\\State
bin/console make:state-processor MyProcessor --namespace-prefix Api\\State
```

> [!NOTE] Namespace prefixes passed to the cli command will be relative to the maker.root_namespace
> and **not** the configured API Platform namepace_prefix.
