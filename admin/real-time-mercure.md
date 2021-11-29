# Real-time Updates With Mercure

API Platform Admin support real-time updates by using the [Mercure protocol](https://mercure.rocks).

Updates are received by using the `useMercureSubscription` hook in the `ListGuesser`, `ShowGuesser` and `EditGuesser` components.

To enable Mercure server-side, see the [related documentation](../core/mercure.md).

To receive API updates from the hub, you need to explicitly enable Mercure in the Hydra data provider.
To do so, you can either do it with a prop in the `<HydraAdmin>` component:

```javascript
import { HydraAdmin } from "@api-platform/admin";

export default () => (
  <HydraAdmin entrypoint="https://demo.api-platform.com" mercure={true} />
);
```

Or in the data provider factory:

```javascript
import { hydraDataProvider, fetchHydra } from "@api-platform/admin";
import { parseHydraDocumentation } from "@api-platform/api-doc-parser";

const dataProvider = baseHydraDataProvider({
  entrypoint,
  httpClient: fetchHydra,
  apiDocumentationParser: parseHydraDocumentation,
  mercure: true,
});
```

If you want to disable Mercure, use `false` for the `mercure` value instead.

## Advanced Configuration

The `mercure` value can also takes an object to customize the default configuration.

The object has the following properties:
- `hub`: the URL to your Mercure hub (default value: entrypoint/.well-known/mercure)
- `jwt`: a subscriber JWT to access your Mercure hub (default value: null)
- `topicUrl`: the topic URL of your resources (default value: entrypoint)
