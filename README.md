# MERN Twitter BONUS: Images!

This bonus walkthrough will show you how to enhance your MERN-Twitter project by
adding **images**!

To store images, audio, or other large files in a MongoDB application, you
have three main options:

1. Store the file in the document itself  
   Obviously, this option will only work if the image can fit within MongoDB's
   16MB document size limit, so something you could potentially use, e.g., for a
   profile avatar. To implement this, you would declare the image in your
   Mongoose Schema to be of SchemaType `Buffer`, which maps to the `binData`
   [BSON (i.e., Binary JSON) Type][BSON] in MongoDB.

2. Use GridFS to store the file(s) in MongoDB  
   [GridFS] is MongoDB's built-in convention for storing large files. It
   essentially stores the files by breaking them into "chunks", which can then
   be accessed individually or reassembled. This option is recommended in the
   following cases (none of which are likely to apply in your projects):

   * You don't have access to an adequate alternative filesystem (e.g., AWS)
   * You want to load sections of a large file without loading the whole file
   * You want to automatically sync files and metadata distributed across
     multiple systems

3. Store a reference to the file's location in a filesystem like AWS  
   The most efficient way to store images and other large files will usually be
   to load them into a filesystem like AWS and then store a reference to the
   file's location in your database.

This guide will walk you through the steps to implement the third option. The
walkthrough builds on the basic MERN Twitter project that you should have
already completed. You can accordingly use your own MERN twitter project or
download a working copy at the `Download project` button below. If you download
a fresh project, **don't forget to update the __backend__ __.env__ file with
your MongoDB credentials!**

Proceed to "Phase 1: Setting Up AWS" to get started!

[BSON]: https://www.mongodb.com/docs/manual/reference/bson-types/
[GridFS]: https://www.mongodb.com/docs/manual/core/gridfs/
