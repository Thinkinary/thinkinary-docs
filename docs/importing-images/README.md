## Importing Images
Thinkinary principally has two methods of importing images:
1. User's device
2. Unsplash

It's best to create an image provider class to handle uploading of images. see [Image provider](/importing-images/?id=image-provider)

### Importing from device
On the web you can import images by using an input field of typ file 
```html
  <input type="file" accept="images/*" onchange="importImage()" />
```

In flutter you can import the image from gallery or camera
```dart
chooseImage() {
  setState(() {
    file = ImagePicker.pickImage(source: ImageSource.gallery);
  });
}
```

### Importing from Unsplash
You can create a provider class to handle getting photos from unsplash. 
[Unsplash API](https://unsplash.com/documentation) specification permits us to get random photos and 
search photos by key words. See the implementation below:

Typescript class example
```typescript
class UnsplashProvider {
  private apiUrl = 'https://api.unsplash.com';
  private accessKey = ACCESS_KEY;

  constructor() { }
  /**
   * Get random photos from unsplash
  */
  getRandom(count = 9): Promise<UnSplashPhoto[]> {
    return fetch(`${this.apiUrl}/photos/random?client_id=${this.accessKey}&count=${count}&orientation=landscape`, {
      method: 'GET',
      headers: {
        'Accept-Version': 'v1',
      }
    }).then(response => response.json());
  }
  /*
  * Search photos
  * */
  searchPhotos(query, count = 9): Promise<UnSplashSearchResults> {
    return fetch(`${this.apiUrl}/search/photos/?query=${query}&client_id=${this.accessKey}&per_page=${count}&orientation=landscape`, {
      method: 'GET',
      headers: {
        'Accept-Version': 'v1',
      }
    }).then(response => response.json());
  }
}

```

Dart class example
```dart
class UnsplashProvider {
  private apiUrl = 'https://api.unsplash.com';
  private accessKey = ACCESS_KEY;
  /**
   * Get random photos from unsplash
  */
  Future<UnSplashPhoto[]> getRandom(int count = 9) {
    return http.get("${this.apiUrl}/photos/random?client_id=${this.accessKey}&count=${count}&orientation=landscape", {
      headers: { 'Accept-Version': 'v1'}
    }).then(response => response.json());
  }
  /*
  * Search photos
  * */
  Future<UnSplashSearchResults> searchPhotos(String query, int count = 9) {
    return http.get(`${this.apiUrl}/search/photos/?query=${query}&client_id=${this.accessKey}&per_page=${count}&orientation=landscape`, {
      headers: {
        'Accept-Version': 'v1',
      }
    }).then(response => response.json());
  }
}
```

`getRandom()` method returns an array of the [UnSplashPhoto](/importing-images/?id=unsplashphoto) interface and the `searchPhotos()` 
method returns the [UnSplashSearchResults](/importing-images/?id=unsplashsearchresults) interface.

#### Uploading to Firebase
When a user selects an unsplash photo, we have to upload this photo to 
firebase cloud storage.First we will use a cloud function endpoint to upload this image to firebase, so in the front end 
application we have to send an http(POST) request with the unsplash photo url as the body of the request 
to the cloud function endpoint.This endpoint will return the new url as response. 
we have to format this image url so we can access via Imagekit enpoint. 
See [Accessing images through imagekit url endpoint](/importing-images/?id=accessing-images-through-imagekitio-url-endpoint).

The cloud function (Node.js)
```javascript
  const fetch = require("node-fetch");
  const functions = require("firebase-functions");
  var remoteimageurl = "https://example.com/images/photo.jpg";
  
  module.exports = functions.onRequest((request, response) => {
      const {fileName} = request.body;
      const imagePath = `images/${fileName}`;
      fetch(remoteimageurl).then(res => {
          return res.blob();
        }).then(blob => {
            //uploading blob to firebase storage
          firebase.storage().ref(imagePath).put(blob).then(function(snapshot) {
            return snapshot.ref.getDownloadURL()
          }).then(url => {
           response.json({url}) 
          }) 
        }).catch(error => {
          response.status(500).json({error});
        });
});
```

### Image Provider 
This section describes Thinkinary's standard of storing and processing images. By default we'll use firebase
to store image. The older method was to upload images directly to imageKit. In the two
cases, [Imakgekit](https://imagekit.io) is used to process the images.

#### Firebase storage
This is default method of saving and processing images. All thinkinary's images will be stored in 
firebase. First the imported image is uploaded to firebase storage. All image uploads must be uploaded in
the images directory.

Example in typescript
```typescript
const fileName = "example.png";
      const imagePath = `images/${fileName}`;
 const uploadTask = firebase.storage().ref(imagePath).put(blob);
```

In dart
```dart
 var fileName = "example.png";
 var imagePath = "images/${fileName}";
 StorageReference storageReference = FirebaseStorage.instance    
       .ref(imagePath);   
 StorageUploadTask uploadTask = storageReference.putFile(_image);
```
##### Accessing images through ImageKit.io URL-endpoint
After the upload is complete, we have to change the download url so we can access through imagekit's url endpoint .The portion of the download url with 
`https://firebasestorage.googleapis.com` is replaced with ` https://ik.imagekit.io/thinkinary`

 Example in typescript
```typescript
 uploadTask.snapshot.ref.getDownloadURL().then((downloadURL) => {
    return downloadURL.toString().replace("https://firebasestorage.googleapis.com","https://ik.imagekit.io/thinkinary");
  });
```

Similarly in dart
```dart
 storageReference.getDownloadURL().then((fileURL) {    
      setState(() {    
        _uploadedFileURL = fileURLdownloadURL.toString().replaceAll("https://firebasestorage.googleapis.com","https://ik.imagekit.io/thinkinary");    
      });
 });
```



#### ImageKit Storage (Deprecated)
This is class that handles uploading to imagekit CDN. The image cdn in structured with 3 directories
and some files in the root directory;

```
.
├── book_covers/
├── article_images/
├── profile_images/
├── docs-image.png
├── logo-name_WqDJPaAvb.jpg
├── placeholder_s5F47pbrD3I.png
└── logo_8f-0ly4nN.png
```

Depending on the context, image uploads shall be in one of the folders; book_cover, articles_images and profile_images.
We can create an enumarator for these paths.
```typescript
enum ImageKitFolders {
  BOOK_COVERS = "/book_covers/",
  ARTICLE_IMAGES = "/article_images/",
  PROFILE_IMAGES = "/profile_images/"
}
```
The end point used to upload images is `https://upload.imagekit.io/api/v1/files/upload`

##### Authentication
Before files can be uploaded, Our Image CDN requires us to have a signature, so we need to get a signature
before each upload

Typescript
```typescript
   getSignature(): Promise<string> {
     return fetch("https://thinkinary-proxy-jade.now.sh/api/auth", {
         method: "GET"
    })
   .then(response => response.json());
}
```

Dart
```dart
Future<string> getSignature() {
  return http.get("https://thinkinary-proxy-jade.now.sh/api/auth");
}
```



##### The Upload function
The upload function takes three paramaters, 
* `file` the image blob or base64 object
* `fileName` the filename of the image, can be custom filename
* `folder` the folder to b uploaded to

```typescript
async upload(file, fileName, folder: ImageKitFolders): Promise<ImageKitResponse> {
    const fd = new FormData();
    fd.append("publicKey", PUBLIC_KEY);
    fd.append("file", file);
    fd.append("fileName", fileName);
    fd.append("folder", folder);
    fd.append("useUniqueFileName", "false");
    // get auth signature and tokens
    const auth = await this.getSignature();
    fd.append("signature", auth.signature || "");
    fd.append("expire", auth.expire || 0);
    fd.append("token", auth.token);
    // upload images
    return ajax({
      url : this.url,
      method : "POST",
      mimeType : "multipart/form-data",
      dataType : "json",
      data : fd,
      processData : false,
      contentType : false,
    });
  }
```

Dart
```dart
Future<ImageKitResponse> upload(File file, String fileName, ImageKitFolders folder): async {
   final auth = await this.getSignature();
   return http.post(this.url, body: {
       "file": base64Encode(file.readAsBytesSync());,
       "folder", folder,
       "name": fileName,
       "publicKey", PUBLIC_KEY,
       "useUniqueFileName", "false",
       "signature", auth.signature || "",
       "expire", auth.expire || 0,
       "token", auth.token
     })
}
```

The upload function returns and object of type [ImageKitResponse](/importing-images/?id=imagekitresponse)

##### Typescript Class
```typescript
class ImageProvider {
  url = "https://upload.imagekit.io/api/v1/files/upload";

  getSignature(): Promise<string> {
    return fetch("https://thinkinary-proxy-jade.now.sh/api/auth", {
        method: "GET"
    })
    .then(response => response.json());
  }

  async upload(file, fileName, folder: ImageKitFolders): Promise<ImageKitResponse> {
      const fd = new FormData();
      fd.append("publicKey", PUBLIC_KEY);
      fd.append("file", file);
      fd.append("fileName", fileName);
      fd.append("folder", folder);
      fd.append("useUniqueFileName", "false");
      // get auth signature and tokens
      const auth = await this.getSignature();
      fd.append("signature", auth.signature || "");
      fd.append("expire", auth.expire || 0);
      fd.append("token", auth.token);
      // upload images
      return ajax({
        url : this.url,
        method : "POST",
        mimeType : "multipart/form-data",
        dataType : "json",
        data : fd,
        processData : false,
        contentType : false,
      });
    }
}

```

##### Dart Class

```dart
class ImageProvider {
  String url = "https://upload.imagekit.io/api/v1/files/upload";

  Future<string> getSignature() {
    return http.get("https://thinkinary-proxy-jade.now.sh/api/auth");
  }

  Future<ImageKitResponse> upload(File file, String fileName, ImageKitFolders folder): async {
     final auth = await this.getSignature();
     return http.post(this.url, body: {
         "file": base64Encode(file.readAsBytesSync());,
         "folder", folder,
         "name": fileName,
         "publicKey", PUBLIC_KEY,
         "useUniqueFileName", "false",
         "signature", auth.signature || "",
         "expire", auth.expire || 0,
         "token", auth.token
       })
  }
}
```

##### ImageKit interfaces

##### ImagekitAuth
```typescript
interface ImageKitAuth{
  signature: string;
  expire: number;
  token: string;
}
```
 
##### ImagekitResponse
The interface of the payload oject sent by imageKit on successful image upload
```typescript
interface ImageKitResponse {
  fileId: string;
  name: string;
  url: string;
  thumbnailUrl: string;
  height: number;
  width: number;
  size: number;
  filePath: string;
  tags: string[];
  isPrivateFile: boolean;
  customCoordinates: any;
  fileType: string;
}
```

### Unsplash Interfaces

##### UnSplashPhoto

```typescript
interface UnSplashPhoto {
  id: string;
  created_at: string;
  updated_at: string;
  promoted_at: string;
  width: number;
  height: number;
  color: string;
  description: string;
  alt_description: string;
  urls: {
    raw: string,
    full: string,
    regular: string;
    small: string,
    thumb: string;
  };
  links: {
    self: string,
    html: string,
    download: string,
    download_location: string
  };
  categories: string[];
  likes: number;
  liked_by_user: boolean;
  current_user_collections: string[];
  user: {
    id: string,
    updated_at: string,
    username: string,
    name: string,
    first_name: string
  };
  views: number;
  downloads: number;
}
```

##### UnSplashSearchResults

```typescript
export interface UnSplashSearchResults {
  results: UnSplashPhoto[];
  total: number;
  total_pages: number;
}
```