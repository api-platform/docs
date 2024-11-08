# Uploading Images and Linking to Other Entities

API Platform Core has file uploading support as shown in [Handling File Upload](../core/file-upload/#handling-file-upload.md). With React Admin's excellent [customization capabilities](customizing.md) we can add file and Image support in the admin panel. 

## Decorating your Data Provider to add upload features

Instead of writing your own Data Provider, you can enhance the capabilities of an existing data provider. You can even restrict the feature only for a given resource (entity) . 

```javascript
// addUploadFeature.js

// Retreiving URL and Headers for AJAX request using fetch api . 
const entrypoint = 'ttp://localhost:8000/api';
const fetchHeaders = {'Authorization': `Bearer ${window.localStorage.getItem('token')}`};

const addUploadCapabilities = requestHandler => {
    return (type, resource, params) => {

        // Upload image to image_objects endpoint and link back the image
        if (type === 'CREATE' && resource === 'books') {

            if (params.data.file.rawFile instanceof File) {
                const body = new FormData();
                body.append("file", params.data.file.rawFile);
                return (fetch(`${entrypoint}/image_objects`, {
                    body,
                    headers: new Headers(fetchHeaders),
                    method: "POST"
                }).then(function (response) {
                    return response.json();
                }).then(function (jsondata) {
                    console.log(jsondata['@id']);
                    return jsondata['@id'];
                }).then(function (imageobject) {
                    return requestHandler(type, resource, {
                        ...params,
                        data: {
                            ...params.data,
                            image: imageobject,
                            file: null,// set file to null

                        },
                    })
                }, (reason => console.log(reason))))
            } 
        }
         return requestHandler(type, resource, params);
    };
};

export default addUploadCapabilities;
```

Take a more complete App.js example from [Authentication Docs](authentication-support.md).To enhance the provider with the upload feature, compose addUploadFeature function with the data provider function:

```javascript
import addUploadFeature from '../addUploadFeature';

// ...

const dataProvider = baseDataProvider(entrypoint, fetchHydra, apiDocumentationParser);
// Add your existing dataprovider to have upload feature and use that instead
const uploadCapableDataProvider = addUploadFeature(dataProvider);

// ...

export default props => (
    <HydraAdmin
        apiDocumentationParser={ apiDocumentationParser }
        dataProvider={ uploadCapableDataProvider }
        authProvider={ authProvider }
        entrypoint={ entrypoint }
    />
);
```

The Admin is now capable of handling files. What is now left is to set appropriate Fields and Input for our `Book` resources. Hence we refer back to [Customising](customizing.md) the admin part to change the Create Book View.

```javascript
const BookCreate = props => (
    <CreateGuesser {...props}>
        <InputGuesser source={"title"} />
        <InputGuesser source={"description"} />
        <InputGuesser source={"URL"} />
        <ImageInput source="file" label="Cover" accept="image/*" placeholder={<p>Drop your file here</p>}>
            <ImageField source="src" title="title"/>
        </ImageInput>
    </CreateGuesser>
```

Upon submission, The file is handled on the client side first, and `POST` uploaded to `/image_objects` endpoint. The response is parsed and the image_object `id`  is added back to default request to be made by the admin. 
