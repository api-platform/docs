# Real-time Updates With Mercure

API Platform Admin support real-time updates by using the [Mercure protocol](https://mercure.rocks).

Updates are received by using the `useMercureSubscription` hook in the `ListGuesser`, `ShowGuesser` and `EditGuesser` components.

To enable Mercure server-side, see the [related documentation](../core/mercure.md).

Once enabled, API Platform Admin will automatically detect that Mercure is enabled and will discover the Mercure hub URL by itself.

## Advanced Configuration

If you want to customize the default Mercure configuration, you can either do it with a prop in the `<HydraAdmin>` component:

```javascript
import { HydraAdmin } from "@api-platform/admin";

export default () => (
  <HydraAdmin
    entrypoint="https://demo.api-platform.com"
    mercure={{ hub: "https://mercure.rocks/hub" }}
  />
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
  mercure: { hub: "https://mercure.rocks/hub" },
});
```

The `mercure` object can take the following properties:
- `hub`: the URL to your Mercure hub (default value: null ; when null it will be discovered by using API responses)
- `jwt`: a subscriber JWT to access your Mercure hub (default value: null)
- `topicUrl`: the topic URL of your resources (default value: entrypoint)
