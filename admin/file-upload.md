# Handling File Upload

If you need to handle the file upload in the server part, please follow [the related documentation](../core/file-upload.md).

This documentation assumes you have a `/media_objects` endpoint accepting `multipart/form-data`-encoded data.

To manage the upload in the admin part, you need to customize [the create form](customizing.md#customizing-the-create-form) or [the edit form](customizing.md#customizing-the-edit-form).

Add a [FileInput](https://marmelab.com/react-admin/Inputs.html#fileinput) as a child of the guesser. For example, for the create form:

```js
import {
  HydraAdmin,
  ResourceGuesser,
  CreateGuesser,
  InputGuesser
} from "@api-platform/admin";
import { FileField, FileInput } from "react-admin";

const MediaObjectsCreate = props => (
  <CreateGuesser {...props}>
    <FileInput source="file">
      <FileField source="src" title="title" />
    </FileInput>
  </CreateGuesser>
);

export default () => (
  <HydraAdmin entrypoint="https://demo.api-platform.com">
    <ResourceGuesser name="media_objects" create={MediaObjectsCreate} />
    {/* ... */}
  </HydraAdmin>
);
```

And that's it!
You don't need to decorate the data provider if you are using the Hydra one: it detects that you have used a `FileInput` and uses a `multipart/form-data` request instead of a JSON-LD one.
