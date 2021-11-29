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
  <HydraAdmin entrypoint="https://demo.api-platform.com" mercure={true}>
    <ResourceGuesser name="media_objects" create={MediaObjectsCreate} />
    {/* ... */}
  </HydraAdmin>
);
```

And that's it!
The guessers are able to detect that you have used a `FileInput` and are passing this information to the data provider, through a `hasFileField` field in the `extraInformation` object, itself in the data.
If you are using the Hydra data provider, it uses a `multipart/form-data` request instead of a JSON-LD one.
In the case of the `EditGuesser`, the HTTP method used also becomes a `POST` instead of a `PUT`, to prevent a [PHP bug](https://bugs.php.net/bug.php?id=55815).
