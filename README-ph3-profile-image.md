# MERN Twitter BONUS, Phase 3: Profile Image

Now that your backend is set up to receive images/files, let's give your users
the chance to upload a profile image when they sign up! This will require you to
modify both the backend and the frontend.

## Backend

First, enable your backend to receive a (single) profile image when a user signs
up.

### Public or private?

Before implementing this feature, you need to decide whether you want the
profile image to be _public_ or _private_. Since users presumably want other
users to see their profile images, the profile image should be public, i.e., you
want to store it in the __public__ folder of your S3 bucket. The bucket policy
that you created in Phase 1 will then allow anyone with the link to read it.
(**Remember:** Public upload is recommended for most portfolio project use
cases.)

### Editing your routes

For this use case, you primarily want to modify the `register` route so that it
can receive and store a profile image. First, however, let's make sure that
`profileImageUrl` is part of the user information being passed to the frontend
in other routes.

Open __backend/config/passport.js__ and add `profileImageUrl` to the `userInfo`
that the `loginUser` middleware returns:

```js
// backend/config/passport.js

exports.loginUser = async function(user) {
  const userInfo = {
    _id: user._id,
    username: user.username,
    profileImageUrl: user.profileImageUrl, // <-- ADD THIS LINE
    email: user.email
  };
  // ...
```

You will also want to make sure that your tweet routes return the
`profileImageUrl` as part of the `author` field. Open
__backend/routes/api/tweets.js__. Find the 4 `.populate` instances--one for each
route--that look like this:

```js
  .populate("author", "_id username")
```

and change them to this:

```js
  .populate("author", "_id username profileImageUrl")
```

Next, go to __backend/routes/api/users.js__. Import the appropriate functions
from __awsS3.js__ at the top of the file. Since this feature requires only a
single image file, import `singleFileUpload` and `singleMulterUpload`:

```js
// backend/routes/api/users.js

const { singleFileUpload, singleMulterUpload } = require("../../awsS3");
```

Next, set the `current` route to include `profileImageUrl` in the user
information it returns:

```js
// backend/routes/api/users.js

router.get('/current', restoreUser, (req, res) => {
  // ...
  res.json({
    _id: req.user._id,
    username: req.user.username,
    profileImageUrl: req.user.profileImageUrl, // <- ADD THIS LINE
    email: req.user.email
  });
})
```

As for the `register` route, install `singleMulterUpload` as the first
middleware on the route. It should look for a key of `image`:

```js
// backend/routes/api/users.js

router.post(
  '/register',
  singleMulterUpload("image"), // <-- ADD THIS LINE
  validateRegisterInput,
  // ...
```

Finally, adjust the `register` route so that it uploads any attached file--the
file will be the profile image--to S3 and sets `profileImageUrl` to the result
(which should be the URL on S3). If there is no file attached to the request,
store the URL to your default profile image that you set up in Phase 2. Your
route should now look something like this:

```js
// backend/routes/api/users.js

router.post(
  '/register', 
  singleMulterUpload("image"), 
  validateRegisterInput, 
  async (req, res, next) => {
    // Check for existing user ...

    // Otherwise create a new user
    const profileImageUrl = req.file ?
      await singleFileUpload({ file: req.file, isPublic: true }) :
      DEFAULT_PROFILE_IMAGE_URL;
    const newUser = new User({
      username: req.body.username,
      profileImageUrl,
      email: req.body.email
    });

    // Salt password and save ...
  }
);
```

(Don't forget to go back and define `DEFAULT_PROFILE_IMAGE_URL` at the top of
the file!)

That's it for the backend! Run `npm run dev` in the __backend__ folder to start
your server and await requests!

## Frontend: Displaying a profile image

Let's start by enabling your frontend to display profile images. For the
purposes of this walkthrough, you will just set the tweets to display a user's
profile image before the username.

To achieve this, open __frontend/src/components/Tweets/TweetBox.jsx__ and modify
it to display the author's profile image:

```js
// frontend/src/components/Tweets/TweetBox.jsx

function TweetBox ({ tweet: { text, author }}) {
  const { username, profileImageUrl } = author; // <-- MODIFY THIS LINE
  return (
    <div className="tweet">
      <h3>
        {/* ADD FOLLOWING CODE TO SHOW PROFILE IMAGE */}
        {profileImageUrl ?
          <img className="profile-image" src={profileImageUrl} alt="profile"/> :
          undefined
        }
        {username}
      </h3>
      <p>{text}</p>
    </div>
  );
}
```

Then add some styling in __TweetBox.css__:

```css
{/* frontend/src/components/Tweets/TweetBox.css */}

.profile-image {
  height: 18px;
  margin-right: 5px;
  margin-left: 2px;
}
```

Now when your app displays "All Tweets", it should display the author's profile
image as well! Of course, right now, all the images are the same default image
that you uploaded in Phase 1. Next you will enable users to upload their own
profile images when they sign up.

## Frontend: Getting an image file

To enable users to upload their own profile images, you first need to modify
your `SignupForm` to take in a profile image as well as a username and password.
Begin by declaring a state variable `image` inside `SignupForm`:

```js
// frontend/src/components/SessionForms/SignupForm.jsx

function SignupForm() {
  // ...
  const [image, setImage] = useState(null);
  // ...
}
```

Then add `image` to the object that you pass to `signup` in `handleSubmit`:

```js
// frontend/src/components/SessionForms/SignupForm.jsx

  const handleSubmit = e => {
    e.preventDefault();
    const user = {
      email,
      username,
      image,    // <-- ADD THIS LINE
      password
    };

    dispatch(signup(user));
  }
```

Next, add a `Profile Image` input in the returned JSX, right above the submit
button. Specify the `type` as `"file"`, and only allow it to `accept` files with
a __.jpg__, __.jpeg__, or __.png__ extension:

```js
// frontend/src/components/SessionForms/SignupForm.jsx

      {/* ... */}
      <label>
        Profile Image
        <input type="file" accept=".jpg, .jpeg, .png" onChange={updateFile} />
      </label>
      {/* ... */}
```

Finally, go back up before the `return` statement and add the `updateFile`
function:

```js
// frontend/src/components/SessionForms/SignupForm.jsx

  const updateFile = e => setImage(e.target.files[0]);
```

File inputs are stored in the input object under `files`. Since the `Profile
Image` input only allows for a single file upload--you will see how to allow
multiple files in a bit--the desired file will always be found at `files[0]`.

## Frontend: Sending an image file

Now that you have the file, you need to send it to the backend so it can be
stored with your user's other information. Recall that in the previous section
you modified the call to `signup` in `SignupForm` to include `image`. In
this section, you will modify `signup` (and `jwtFetch`) so that they can
process that `image` appropriately.

Open __frontend/src/store/session.js__ and find the `signup` thunk action
creator. Unfortunately, you cannot send files to your backend using simple JSON;
they are just too big. You will accordingly need to send them as [FormData]
instead. Fortunately, `FormData` is rather straightforward to configure. You
simply create a new `FormData` instance and `append` whatever fields you need.
Add the following code to the beginning of `startSession`:

```js
// frontend/src/store/session.js

const startSession = (userInfo, route) => async dispatch => {
  const { image, username, password, email } = userInfo;
  const formData = new FormData();
  formData.append("username", username);
  formData.append("password", password);
  formData.append("email", email);

  if (image) formData.append("image", image);
  // ...
};
```

**Note that you only want to append `image` if there is an image to append.**

Then make sure that you send your FormData object instead of JSON as the body of
the request:

```js
// frontend/src/store/session.js

const startSession = (userInfo, route) => async dispatch => {
  // ...
  try {  
  const res = await jwtFetch(route, {
    method: "POST",
    body: formData  // <-- CHANGE THIS LINE
  });
  // ...
```

> **Note:** Because you are changing `startSession`, your `login` route will
> also need to use FormData instead of JSON. That's fine! You just need to
> install Multer on the `login` route as well. Although this route won't need to
> process any image files, Multer will also put the FormData text fields under
> `body` in the request. Add it like this:

  ```js
  // backend/routes/api/users.js

  router.post(
    '/login',
    singleMulterUpload(""), // <-- ADD THIS LINE
    validateRegisterInput,
    // ...
  ```

For this version of `startSession` to work, you need to make one more change.
`FormData` uses the `multipart/form-data` encoding, but you cannot set the
`Content-Type` header to `multipart/form-data` manually because it needs to
contain information about the multi-part boundaries. Fortunately, this gets
handled automatically as long as the `Content-Type` header remains empty. You
just have to make sure that your `jwtFetch` function doesn't explicitly set that
header if the body is of type `FormData`.

Go to __frontend/src/store/jwt.js__. Inside `jwtFetch`, change this line

```js
// frontend/src/store/jwt.js

  options.headers["Content-Type"] =
    options.headers["Content-Type"] || "application/json";
```

to this:

```js
// frontend/src/store/jwt.js

  if (!options.headers["Content-Type"] && !(options.body instanceof FormData)) {
    options.headers["Content-Type"] = "application/json";
  }
```

Your app is now ready to send a user's profile image file to the backend for
storage! Try it out by starting up your frontend (`npm start`), signing up as a
new user, and selecting a profile image. Once you are signed in, try writing a
tweet. The image should appear next to your username in the preview!

In the final phase, you will enable users to add images to their tweets.

[FormData]: https://developer.mozilla.org/en-US/docs/Web/API/FormData
