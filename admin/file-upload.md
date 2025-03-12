# Handling File Upload

If you need to handle the file upload in the server part, please follow [the related documentation](../symfony/file-upload.md).

This documentation assumes you have a `/media_objects` endpoint accepting `multipart/form-data`-encoded data.

To manage the upload in the admin part, you need to [customize the guessed create or edit form](./customizing.md#customizing-the-editguesser-and-createguesser).

Add a [FileInput](https://marmelab.com/react-admin/FileInput.html) as a child of the guesser. For example, for the create form:

```js
import {
  HydraAdmin,
  ResourceGuesser,
  CreateGuesser,
} from '@api-platform/admin';
import { FileField, FileInput } from 'react-admin';

const MediaObjectsCreate = () => (
  <CreateGuesser>
    <FileInput source="file">
      <FileField source="src" title="title" />
    </FileInput>
  </CreateGuesser>
);

export const App = () => (
  <HydraAdmin entrypoint="https://demo.api-platform.com">
    <ResourceGuesser name="media_objects" create={MediaObjectsCreate} />
    {/* ... */}
  </HydraAdmin>
);
```

And that's it!
The guessers are able to detect that you have used a `FileInput` and are passing this information to the data provider, through a `hasFileField` field in the `extraInformation` object, itself in the data.
If you are using the Hydra data provider, it uses a `multipart/form-data` request instead of a JSON-LD one.

**Note:** In the case of the `EditGuesser`, the HTTP method used becomes a `POST` instead of a `PUT`, to prevent a [PHP bug](https://bugs.php.net/bug.php?id=55815).
