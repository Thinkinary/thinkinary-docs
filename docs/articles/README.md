## Articles

```typescript
class Article {
  id: string;
  title: string;
  content: string;
  coverImageURL: string;
  tags: string[];
  author: UserModel;     // author of type UserModel
  authorUID: string;
  reads: number;
  comments: Comment[];
  commentsCount: number;
  fileId: string; // imagekit upload file id, used to delete image from imagekit
  thumbnails: {
    small: string;
    medium: string;
    large: string;
    placeholder: string;
  };
  // imageURLS: any[];
  shareUrl: string;
  date: Date;
  urlID: string;
}
```

### Creating articles

#### Overview

The process of adding articles varies for different platforms.
On mobile the user starts by specifying the article title and on the 
web the user is faced directly with an editor. However the case, when a user 
initializes the article (be it by specifying the article title, article body or the cover image), a draft is first created. The draft shall be saved the firebase realtime database
in the order:

```typescript
  {
    user_uid : {
        draft_one_uid : {
          title: "Article one title"
          content: "This is the article's body..."
        },
        draft_two_uid : {
           title: "Article two title"
           content: "This is the article's body..."
        }
      }  
  }
```
Each time the user updates the body of the article, it's a good practice to wait for the user to finish typing before
saving the article to draft. The draft exist till the article is published or submitted for review.

Create an `Article` instance and initialize some properties. 
You can decide to pass the initialization of as constructor props.
 This is what we'll submit to the database. example
```typescript
const article = new Article();
// Specifying the author
article.author = {
    uid: this.user.uid,
    displayName: this.user.displayName,
    email: this.user.email,
    photoURL: this.user.photoURL
};
article.tags = [];          // initialize the article tags as an empty array
article.commentsCount: 0;   // initialize comments count to 0
article.reads: 0;           // initialize article reads count to 0
```

once the user populate the input fields we can update our article object.
```typescript
   article.title = "The IKEA effect..";
   article.content = "This is just a basic principle based on...";
```


#### Importing Images
##### Cover images
There are two ways to import a cover image: importing from device and from unsplash
see [importing images](/importing-images/). It is best to create an image uploader provider that
would be used by article module and profile module to upload images. Check the creating 
image provider guide

Here is a snippet of an upload method in `Typescript`
```typescript
uploadImage(file, name = null) {
    // show loading animation
    const fileName =  name ? name : file.name.split(' ').join('').split('.')[0]; // remove whitespaces from filename
    // Check the image upload guide to
    this.imageProvider.upload(file, fileName, ImageKitFolders.ARTICLE_IMAGES)
      .then((response) => { 
        // patch the article object
        article.fileId = response.fileId; // the uploaded image file Id
        article.coverImageURL = response.url; // the uplaoded image url
        article.thumbnails = {
            large: generateNamedTransformation(res.url, 'large'),
            medium: generateNamedTransformation(res.url, 'medium'),
            small: generateNamedTransformation(res.url, 'small'),
            placeholder: generateNamedTransformation(res.url, 'placeholder'),
        }
      }).catch(err => {
        this.toast.notify('Failed to upload image');
        console.error(err);
      })
      .finally(() => {
        // remove loading animation
      });
  }
```
Check the [importing images](importing-images/) guide to learn how to build a provider for your platform.

Once the image is uploaded, the article object is patched with the response from the image CDN;
* `response.fileId` : The file id of the image. It very is important because it is used to delete an image from the image CDN
* `response.url` : This is the url of the uploaded image.

###### Generating thumbnails
Create a function to generate thumbnails of named transformations.
Thumbnails are used to by the frontend applications on several occasion, for example the small thumbnails are used in list views,
the large thumbnails for the article read view.

In typescript
```typescript
declare type NamedTransformationType = 'small' | 'medium' | 'large' | 'placeholder';

export function generateNamedTransformation(url: string, type: NamedTransformationType) {
  return `${url}?tr=n-${type}`;
}
```

In dart

```dart
// type can only have the values: 'small' , 'medium' , 'large' , 'placeholder';
 
String generateNamedTransformation(String url, String type) {
  return "${url}?tr=n-${type}";
}
```

##### Editor images
Sometimes the user might want to add images through the article editor. most editors encode the 
image to a base64 string, this behaviour is definitely not favorable. so We need to override the default
behaviour and upload the image to the image CDN and return an image url to the editor.


### Saving articles
Once the user is done typing and clicks on complete, Now he is presented a screen to add tags or skip to publish.

#### Adding Tags
If tags are added by the user, add the tags to the article object. Here is a code snippet in typescript
Do to do the same implementation in any targeted language.

```typescript
addTag(tag: string) {
    // exit function if user adding more that 60 tags
    if (this.selectedTags.length > 10) {
      this.toast.notify('Maximum number of tags reached');
      return;
    };
    value = tag.indexOf('#') > -1 ? tag : '#' + tag; // check for the presence of '#', add it if it doesn't exist
    // remove white spaces
    value = value.trim().split(' ').join('');
    // add tags to article arrray earlier initialized
    article.tags.push(value);
  }
```

#### The Author
A user is required to be authenticated to submit an article. 
You should get the user's data with [custom claims](https://firebase.google.com/docs/auth/admin/custom-claims).
Thinkinary currently has two user claims, `Admin` and `Editor`. These claims are used to 
determine weather to publish the article directly or submit it for review.
Check the firebase user claims docs for your platform.

#### Generating the URL ID
The url ID is a hyphenated article's title, passed as a parameter in the article's url which is used to fetch
 the article from the database. for example `https://thinkinary.com/article/how-to-cope-with-cyberbullying`. \
 The url ID is the parameter `how-to-cope-with-cyberbullying`. This is used instead of the database primary for 
 readability.
 
 In typescript
 ```typescript
function getUrlID(title: string): string {
   return title.replace(/[^a-zA-Z0-9]/g, ' ')    // replace non-alphanumeric characters with whitespace
           .toLowerCase()                        // convert string to lowercase
           .replace(/\s+/g, ' ')                 // replace multiple whitespaces with single whitespace
           .trim()                               // trim string, i.e remove whitespaces from start and end of  the string
           .replace(/\s/g, '-');                 // replace single whitespaces with hyphen
}
```

 In dart
 ```dart
String getUrlID(String title) {
   return title.replaceAll(/[^a-zA-Z0-9]/g, ' ')
           .toLowerCase()
           .replaceAll(/\s+/g, ' ')
           .trim()
           .replaceAll(/\s/g, '-');
}
```

#### Adding to Database
Now let's review article payload
```typescript
// article
{
   title: "The IKEA effect..",
   urlID: getUrlID("The IKEA effect.."),    // returns: the-ikea-effect
   content: "Article's content in markdown",
   author: {...},
   authorUID: this.user.uid,
   commentsCount: 0,
   reads: 0,
   tags: ["#Positivity", "#2020"],
   coverImageURL: "https://ik.imagekit.io/upload/image.png",
   thumbnails: {...},
   fileId: "ikst34ERERFd",
   date: new Date();
}
```

However it's important to check if the coverImageURL exist (ie. it was explicitly imported by the user). 
If it doesn't exist, you should assign thinkinary's default article image placeholder
```typescript
if (!article.coverImageURL) {
  article.coverImageURL = "https://ik.imagekit.io/thinkinary/placeholder_s5F47pbrD3I.png";
}
```

Now depending on the user custom claims: if the user has the custom claim of `Admin` or `Editor`. \
Add the article to the `articles` collection.

```typescript
// show loader
this.db.addArticle(article)
   .then((itemRef) => {
     // add article to algolia index
     fetch('https://thinkinary-proxy-jade.now.sh/api/articles/?type=add', {
         method: 'POST',
         body: JSON.stringify({id: itemRef.id, objectID: itemRef.id,  ...article}) 
     })
     // route to article complete page
     this.router.navigate(['/article/' + urlId]);
     // In flutter
     // Navigator.push(ArticleScreen(urlId))
   })
   .catch((err) => {
      // show toast with error message
      this.toast.notify(err.message);
   })
   .finally(() => {
      // remove loader
      this.loading = false;
   });
```

When an article was added to articles collection succesfully.\
send a request to the `https://thinkinary-proxy-jade.now.sh/api/articles/?type=add` endpoint
to add the article to algolia's search index.\
You can use the http package in flutter to achieve this.
```dart
Future<http.Response> createAlbum(String title) {
  return http.post(
    'https://thinkinary-proxy-jade.now.sh/api/articles/?type=add',
    headers: <String, String>{
      'Content-Type': 'application/json; charset=UTF-8',
    },
    body: jsonEncode(<String, String>{id: itemRef.id, objectID: itemRef.id,  ...article}),
  );
}
```

If the user is neither an Admin or an Editor, add the article to the `unverifiedArticles` collection,
then redirect the user to the article submitted screen/view.