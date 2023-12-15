# MERN Twitter BONUS, Phase 4: Images

In this final phase of the walkthrough, you will enable users to upload multiple
images to their tweets. Once again, you will need to modify both the backend and
the frontend.

## Backend

On the backend, you will need to modify the `POST` routes for `tweets` to upload
the images to S3 and save the resulting links to the database.

Open __backend/routes/api/tweets.js__ and load `multipleFilesUpload` and
`multipleMulterUpload` from __awsS3.js__:

```js
// backend/routes/api/tweets.js

const { multipleFilesUpload, multipleMulterUpload } = require("../../awsS3");
```

In the `post` route, insert `multipleMulterUpload("images")` as the first
middleware. Then in the callback, call `multipleFilesUpload` to upload the
images to S3. This will return an array of URLs. Store that array in the
database.

Your code should look something like this:

```js
// backend/routes/api/tweets.js

router.post(
  '/',
  multipleMulterUpload("images"), // <-- ADD THIS LINE
  requireUser, 
  validateTweetInput, 
  async (req, res, next) => {
    // ADD THE LINE BELOW
    const imageUrls = await multipleFilesUpload({ files: req.files, isPublic: true });
    try {
      const newTweet = new Tweet({
        text: req.body.text,
        imageUrls,              // <-- ADD THIS LINE
        author: req.user._id
      });

      let tweet = await newTweet.save();
      tweet = await tweet.populate('author', '_id username profileImageUrl');
      return res.json(tweet);
    }
    catch(err) {
      next(err);
    }
  }
);
```

## Frontend: Getting the images to send

On the frontend, you first need to modify `TweetCompose` to receive images.

Open __frontend/src/components/Tweets/TweetCompose.jsx__. Declare `images` and
`imageUrls` as state array variables:

```js
// frontend/src/components/Tweets/TweetCompose.jsx

  const [images, setImages] = useState([]);
  const [imageUrls, setImageUrls] = useState([]);
```

`images` will store the image files that you will send to the backend;
`imageUrls` will store temporary URLs for those images for use in your preview
functionality.

Next, modify `handleSubmit` to send `images` to `composeTweet` as well as
`text`. Also, remember to reset `images` and `imageUrls` at the end
of`handleSubmit`:

```js
// frontend/src/components/Tweets/TweetCompose.jsx

const handleSubmit = e => {
    e.preventDefault();
    dispatch(composeTweet(text, images)); // <-- MODIFY THIS LINE
    setImages([]);                        // <-- ADD THIS LINE
    setImageUrls([]);                     // <-- ADD THIS LINE
    setText('');
  };
```

Then add an `Images to Upload` field to your form after the `textarea` input:

```js
// frontend/src/components/Tweets/TweetCompose.jsx

        <label>
          Images to Upload
          <input
            type="file"
            accept=".jpg, .jpeg, .png"
            multiple
            onChange={updateFiles} />
        </label>
```

This `input` has a significant difference from the avatar `input` that you
created in the Phase 3: it includes the `multiple` attribute. Adding this
attribute is all you need to do to enable your input to accept more than one
file. Note, however, that all the files need to be selected at the same time:
each time you click `Choose Files`, it will reset the `files` field. (Selecting
multiple files at once can be done, for instance, by clicking a file and then
clicking on a second file while holding down <Shift>.)

Also modify the `Tweet Preview` to display the preview if either text or an
image is present. You will need to pass `imageUrls` as part of the `tweet` as
well:

```js
// frontend/src/components/Tweets/TweetCompose.jsx

      <div className="tweet-preview">
        <h3>Tweet Preview</h3>
        {(text || imageUrls.length !== 0) ?                  // <-- MODIFY THIS LINE
            <TweetBox tweet={{text, author, imageUrls}} /> : // <-- MODIFY THIS LINE
            undefined}
      </div>
```

Now add an `updateFiles` function before the `return`. This function needs to
set `images`, but it also needs to generate and set temporary `imageUrls` for
the preview. To accomplish the latter, create a [`FileReader`] instance for each
image in `files` and invoke [`readAsDataURL`] with the image `file` passed as
the argument. This will trigger an async action. Define an `onload` property on
the `FileReader` instance that points to a callback that will run after
`readAsDataURL` completes. In that callback, store `fileReader.result`--i.e.,
the temporary URL--at the appropriate index of a URL array. If all the files
have finished generating their URLs, then `setImageUrls` to the URL array.

This is a bit tricky; if you like a challenge, try writing `updateFiles` without
looking at the code below. A tip: `e.target.files` is an array-like `FileList`
rather than an actual `Array`. (For more on the `FileList` type, see the [MDN
docs][filelist].) You will accordingly need to cast it to an `Array` before you
can run methods like `forEach` on it.

Ultimately, your code should look something like this:

```js
// frontend/src/components/Tweets/TweetCompose.jsx

  const updateFiles = async e => {
    const files = e.target.files;
    setImages(files);
    if (files.length !== 0) {
      let filesLoaded = 0;
      const urls = [];
      Array.from(files).forEach((file, index) => {
        const fileReader = new FileReader();
        fileReader.readAsDataURL(file);
        fileReader.onload = () => {
          urls[index] = fileReader.result;
          if (++filesLoaded === files.length) 
            setImageUrls(urls);
        }
      });
    }
    else setImageUrls([]);
  }
```

> **Note:** The above code completely replaces the images / image URLs each time
> a user selects new images. Feel free to implement an alternative behavior
> (e.g., appending new images to ones already selected) if you prefer.

Top off your additions by adding some styling to __TweetCompose.css__:

```css
/* frontend/src/components/Tweets/TweetCompose.css */

label {
  display: block;
  margin-top: 5px;
}

input[type=file] {
  margin-left: 5px;
}
```

[`FileReader`]: https://developer.mozilla.org/en-US/docs/Web/API/FileReader
[`readAsDataURL`]: https://developer.mozilla.org/en-US/docs/Web/API/FileReader/readAsDataURL
[filelist]: https://developer.mozilla.org/en-US/docs/Web/API/FileList

### Clearing the image filenames on submit

Your `handleSubmit` function currently clears the `images` and `imageUrls` state
variables, but the images are still stored in `files`. As a result, the `Choose
Files` field on your form will not reset when you submit. It will look like you
still have the same images selected, but because you haven't stored these files
in `images` again, submitting the form will **NOT** include those former images.
(Remember that you only store images in `images` when the file input is
updated.)

To reset `files`, you need to set the input element's `value` to `null`.
[`useRef`] is a great way to do this. Import [`useRef`] along with `useState`
and `useEffect`, and declare the reference like this:

```js
// frontend/src/components/Tweets/TweetCompose.js

const fileRef = useRef(null);
```

Assign this reference to `ref` when you declare the input element in the form
itself:

```js
// frontend/src/components/Tweets/TweetCompose.js

        <label>
          Images to Upload
          <input
            type="file"
            ref={fileRef}       // <-- ADD THIS LINE
            accept=".jpg, .jpeg, .png"
            multiple
            onChange={updateFiles} />
        </label>
```

This will store a link to the element in `fileRef.current`. You can then use
this reference to update the value in your `handleSubmit` function:

```js
fileRef.current.value = null;
```

[`useRef`]: https://reactjs.org/docs/hooks-reference.html#useref

## Frontend: Sending the images

To send the images, open __frontend/src/store/tweets.js__ and adjust
`composeTweet` to send the images to the backend as `FormData`:

```js
// frontend/src/store/tweets.js

export const composeTweet = (text, images) => async dispatch => {
  const formData = new FormData();
  formData.append("text", text);
  Array.from(images).forEach(image => formData.append("images", image));
  try {
    const res = await jwtFetch('/api/tweets/', {
      method: 'POST',
      body: formData // <-- DON'T FORGET TO CHANGE THIS LINE TOO
    });
    // ...
```

Note that if you did not already change `jwtFetch` to leave the `Content-Type`
header blank when the body is `FormData` in Phase 3, you will need to do that
now for `composeTweet` to work.

## Frontend: Displaying images

The final step is to display the images in your tweets!

Open __TweetBox.js__. Start by adding `imageUrls` as one of the deconstructed
parameters alongside `text` and `author`. Map over the imageUrls and create an
`img` tag for each one. Then display the at the bottom of the `tweet`:

```js
// frontend/src/components/Tweets/TweetBox.js

function TweetBox ({ tweet: { text, author, imageUrls }}) { // <-- MODIFY THIS LINE
  const { username, profileImageUrl } = author;
  // ADD THE IMAGE PROCESSING BELOW
  const images = imageUrls?.map((url, index) => {
    return <img className="tweet-image" key ={url} src={url} alt={`tweetImage${index}`} />
  });
  return (
    <div className="tweet">
      <h3>
        {profileImageUrl ? 
          <img className="profile-image" src={profileImageUrl} alt="profile"/> :
          undefined
        }
        {username}
      </h3>
      <p>{text}</p>
      {images}      {/* ADD THIS LINE */}
    </div>
  );
}
```

Finally, add the following styling in __TweetBox.css__:

```css
/* frontend/src/components/Tweets/TweetBox.css */

.tweet-image {
  height: 100px;
  margin: 5px;
}
```

That's it! Test your new functionality by writing some tweets with images!
