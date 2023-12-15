# MERN Twitter BONUS, Phase 2: Backend

In this phase, you will set up your backend to receive a file from the frontend,
store it in S3, and save the resulting link/key to the database.

## Set up your AWS credentials in your app

You can supply your app with the AWS user credentials you created at the end of
Phase 1 through environment variables. You'll also need to supply the [AWS
region] (e.g., `us-east-1`). Add the following to your __backend/.env__ file:

```plaintext
AWS_REGION=<Your region, e.g., 'us-east-1'>
AWS_ACCESS_KEY_ID=<Your access key id here>
AWS_SECRET_ACCESS_KEY=<Your secret access key here>
```

The `aws-sdk` package--which you will set up in a moment--will enable AWS to
grab your region and credentials from these environment variables. **Remember
that when you deploy to Render.com or Heroku, you will need to set these
environment variables there as well.**

> You can read more about how AWS uses environment variables
> [here][aws-environment-variables].

**MAKE SURE TO GITIGNORE YOUR __.ENV__ FILE.**

## Update your models and documents

MongoDB itself doesn't really care what information a database document contains
or how it is organized: if you want to add a field to a particular document, you
just add it. It will be helpful, however, to update your Mongoose models to
incorporate images.

To start, adjust your models to store a single profile image URL in your User
model and an array of image URLs in your Tweet model:

```js
// backend/models/User.js

const userSchema = new Schema({
  username: {
    type: String,
    required: true
  },
  // ADD profileImageUrl
  profileImageUrl: {
    type: String,
    required: true
  },
  // ...
```

```js
// backend/models/Tweet.js

const tweetSchema = new Schema({
  author: {
    type: Schema.Types.ObjectId,
    ref: 'User'
  },
  // ADD imageUrls
  imageUrls: {
    type: [String],
    required: false
  },
  // ...
```

Updating the model does not update the documents in your database. To update the
existing documents, you will need to write a migration that goes through and
makes the necessary edits. You can think of this migration as another seed file;
you will build it in the same way.

You will first need a default profile image to use for users who do not supply
one. If you need an image to use for this purpose, you can download a free one
that does not require attribution from [Pixabay]. Then upload your file to your
S3 bucket:

1. Go to your [S3 buckets page] and select the bucket you created in Phase 1.
2. If you do not already have a __public__ folder in the bucket, click `Create
   folder` and add one, accepting all the folder defaults.
3. Back on the bucket page, click on the (newly created) __public__ folder.
4. Inside __public__, click the orange `Upload` button, then `Add files`.
5. Select your default profile image and upload it.
6. Back inside your __public__ folder, click on the image file you just
   uploaded.
7. Copy the `Object URL` that appears in blue. This is the URL you will use to
   access this image.

Next, in your __backend/seeders__ route, create an __images.js__ file and
copy in the following code:

```js
// backend/seeders/images.js

const mongoose = require("mongoose");
const { mongoURI: db } = require('../config/keys.js');
const User = require('../models/User');
const Tweet = require('../models/Tweet');

const DEFAULT_PROFILE_IMAGE_URL = 'YOUR-URL-HERE'; // <- Insert the S3 URL that you copied above here

// Connect to database
mongoose
  .connect(db, { useNewUrlParser: true })
  .then(() => {
    console.log('Connected to MongoDB successfully');
    initializeImages();
  })
  .catch(err => {
    console.error(err.stack);
    process.exit(1);
  });

// Initialize image fields in db
const initializeImages = async () => {
  console.log("Initializing profile avatars...");
  await User.updateMany({}, { profileImageUrl: DEFAULT_PROFILE_IMAGE_URL });
    
  console.log("Initializing Tweet image URLs...");
  await Tweet.updateMany({}, { imageUrls: [] });

  console.log("Done!");
  mongoose.disconnect();
}
```

**Don't forget to paste your default S3 URL as the PROFILE_IMAGE_URL.**

[`updateMany`] will update all documents that meet the condition set in the
first argument (here, `{}`) with the document information provided in the second
argument (e.g., `{ imageUrls: [] }`). By leaving the test condition blank
(`{}`), you assure that every document will be updated.

Execute this update by running the following command in the __backend__
directory:

```sh
dotenv node seeders/images.js
```

If everything runs correctly, you should now be able to see the updated fields
in your Mongo database collections on [Atlas].

[S3 buckets page]: https://s3.console.aws.amazon.com/s3/home
[`updateMany`]: https://mongoosejs.com/docs/api.html#model_Model-updateMany
[Pixabay]: https://pixabay.com/vectors/blank-profile-picture-mystery-man-973460/
[Atlas]: https://www.mongodb.com/atlas/database

## Install necessary packages

Next, `npm install` the following packages in your __backend__ folder:

- [multer]
- [@aws-sdk/client-s3]
- [@aws-sdk/lib-storage]

## Set up AWS S3 v3

Make a file called __awsS3.js__ at the root of your __backend__ directory and
copy in the following content:

```javascript
// backend/awsS3.js

const { S3Client } = require("@aws-sdk/client-s3");
const { Upload } = require ("@aws-sdk/lib-storage");
const multer = require("multer");
const NAME_OF_BUCKET = "<NAME-OF-YOUR-BUCKET>"; // <-- Use your bucket name here

module.exports = {
  // You will list your exports here
};
```

**Make sure that you replace the `<NAME-OF-YOUR-BUCKET>` at the top of the
__awsS3.js__ file with the name of your bucket.**

This file will house all of your AWS-related functions. You will build it out as
you go along.

### Enable uploading of images to S3

Start by creating a function in __awsS3.js__ to upload a single file to S3:

```js
// backend/awsS3.js

const singleFileUpload = async ({ file, isPublic = false }) => {
  const { originalname, buffer } = file;
  const path = require("path");

  // Set the name of the file in your S3 bucket to the date in ms plus the
  // extension name.
  const Key = new Date().getTime().toString() + path.extname(originalname);
  const params = {
    Bucket: NAME_OF_BUCKET,
    Key: isPublic ? `public/${Key}` : Key,
    Body: buffer
  };
  const client = new S3Client({});

  try {
    const parallelUploadS3 = new Upload({ client, params });
    parallelUploadS3.on("httpUploadProgress", (progress) =>
      console.log(progress)
    );
    const result = await parallelUploadS3.done();

    // Return the link if public. If private, return the name of the file in your
    // S3 bucket as the key in your database for subsequent retrieval.
    return isPublic ? result.Location : result.Key;
  } catch (err) {
    console.log(err);
  }
};
```

Don't forget to add it to your exports:

```js
module.exports = {
  singleFileUpload
};
```

As the name suggests, this function takes in a single file and uploads it to
your S3 bucket. A few points are worth noting:

1. The function takes in an options hash that with an `isPublic` key that defaults
   to `false`: if you want to make something public, you must explicitly say so.

2. Every file in your bucket will need a unique name. The function takes care of
   this by turning a timestamp (plus original extension) into the name of the
   stored file:

   ```js
   const Key = new Date().getTime().toString() + path.extname(originalname);
   ```

3. If a file should be public, you need to add it to the __public__ folder. To
   add a file to the __public__ folder, you must prepend `public/` to the
   **`Key`**, not append it to the **`Bucket`** name. This will always be the
   case. You never modify the bucket name; just add any desired path to the
   key/filename and AWS will interpret it as the directory structure. (If a
   __public__ folder does not yet exist in the bucket, uploading a file
   beginning with `public/` will create it.)

   > __Note:__ There is nothing magical about naming a folder __public__. Items
   > in the __public__ folder are publicly available because of the bucket
   > policy you created in Phase 1. If you made your whole bucket public in
   > Phase 1, then you do not need to create or use a __public__ folder.

4. The other point where the private/public distinction comes into play is with
   the return value. Since a public file is publicly accessible, you can simply
   return a link to its location in your S3 bucket (`result.Location`).
   Accessing private files, in contrast, will require your IAM user's AWS
   credentials, and you don't want to store those in the database. So for
   private files, you just return the key/filename (`result.Key`).

Sometimes you will want to upload more than a single file, so go ahead and add a
`multipleFilesUpload` function as well:

```js
// backend/awsS3.js

const multipleFilesUpload = async ({files, isPublic = false}) => {
  return await Promise.all(
    files.map((file) => {
      return singleFileUpload({file, isPublic});
    })
  );
};

module.exports = {
  singleFileUpload,
  multipleFilesUpload
};
```

### Enable retrieving of private files

As noted above, you will need the IAM user's credentials to retrieve private
files. Another AWS-SDK package--[@aws-sdk/s3-request-presigner]--includes a
helpful function, `getSignedUrl`, that will generate a link with temporary and
limited authentication built in. You won't need it for this
walkthrough--everything is public--but if you ever need to write a function to
return a link authorized for getting/reading the specified object, this is how
you would do it.

First, `npm install` the `@aws-sdk/s3-request-presigner` package and import `getSignedUrl`. You'll also need to import `GetObjectCommand` from `@aws-sdk/client-s3`. Then construct this function:

```js
// backend/awsS3.js

const retrievePrivateFileUrl = async (key) => {
  let fileUrl;
  const client = new S3Client({});

  if (key) {
    const command = new GetObjectCommand({
      Bucket: NAME_OF_BUCKET,
      Key: key
    });
    fileUrl = await getSignedUrl(client, command);
  }

  return fileUrl || key;
};

module.exports = {
  singleFileUpload,
  multipleFilesUpload,
  retrievePrivateFileUrl
};
```

(The `@aws-sdk/s3-request-presigner` package uses your IAM credentials in the
__.env__ file to authorize this call to `getSignedUrl`.)

## Multer

Your backend now has the ability to store files in S3, but how does it get the
files in the first place? Files, which are too big to pass in a single request,
will arrive in requests with content type `multipart/form-data`.
[Multer][multer] is Node.js middleware that takes `multipart/form-data`
requests, puts the text fields into a `body` object on the request, and
reassembles the file(s) under a `file` or `files` key in the request.

### Multer setup

You can set up Multer in your __awsS3.js__ file like this:

```js
// backend/awsS3.js

const storage = multer.memoryStorage({
  destination: function (req, file, callback) {
    callback(null, "");
  },
});

const singleMulterUpload = (nameOfKey) =>
  multer({ storage: storage }).single(nameOfKey);
const multipleMulterUpload = (nameOfKey) =>
  multer({ storage: storage }).array(nameOfKey);

module.exports = {
  singleFileUpload,
  multipleFilesUpload,
  retrievePrivateFile,
  singleMulterUpload,
  multipleMulterUpload
};
```

In `singleMulterUpload` and `multipleMulterUpload`, **`nameOfKey` must match the
name under which you stored the data on the frontend.** Note that Multer is
strict when processing fields: it will throw an `Unexpected Field` error if
there is a non-text field stored under a name other than what was passed as
`nameOfKey`. You can't, for instance, pass a request with both a single `image`
and an array of `images` to `singleMulterUpload("image")`; `images` will be an
`Unexpected Field`.

Now you just need to install the appropriate Multer instance on routes where you
want to receive files/images! You'll take care of that in the next two phases.

[multer]: https://www.npmjs.com/package/multer
[@aws-sdk/client-s3]: https://www.npmjs.com/package/@aws-sdk/@aws-sdk/client-s3
[@aws-sdk/lib-storage]: https://www.npmjs.com/package/@aws-sdk/lib-storage
[@aws-sdk/s3-request-presigner]: https://www.npmjs.com/package/@aws-sdk/s3-request-presigner
[aws-environment-variables]: https://docs.aws.amazon.com/sdkref/latest/guide/environment-variables.html
[AWS
region]: https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/setting-region.html
