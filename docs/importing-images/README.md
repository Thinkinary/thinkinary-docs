## Importing Images
Thinkinary principally has two methods of importing images:
1. User's device
2. Unsplash

It's best to create an image provider class to handle uploading of images. see [Image provider](/importing-images/?id=image-provider)

### Importing form device
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

### Importing form Unsplash
You can create a provider class to handle getting photos from unsplash

#### Typescript class

```typescript
class UnsplashProvider {
  private apiUrl = 'https://api.unsplash.com';
  private accessKey = ACCESS_KEY;

  constructor() { }

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

#### Dart class

```typescript
class UnsplashProvider {
  private apiUrl = 'https://api.unsplash.com';
  private accessKey = ACCESS_KEY;

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

`getRandom()` method returns an array of [UnSplashPhoto interface](/importing-images/?id=unsplashphoto) and the `searchPhotos()` 
method returns the [UnSplashSearchResults](/importing-images/?id=unsplashsearchresults) interface

### Image Provider
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

#### Authentication
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



#### The Upload function
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

#### Typescript Class
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

#### Dart Class

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

#### ImageKit interfaces

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

#### Unsplash Interfaces

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