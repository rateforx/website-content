---
title: Handling file uploads in forms using Express
description: "How to manage storing and handling files uploaded via forms, in Express"
date: 2018-09-21T07:00:00+02:00
tags: express
tags_weight: 5
path: express-forms-files
---

This is an example of an HTML form that allows a user to upload a file:

```html
<form method="POST" action="/submit-form">
  <input type="file" name="document" />
  <input type="submit" />
</form>
```

When the user presses the submit button, the browser will automatically make a `POST` request to the `/submit-form` URL on the same origin of the page. The browser sends the data contained, not encoded as as a normal form `application/x-www-form-urlencoded` , but as `multipart/form-data`.

Server-side, handling multipart data can be tricky and error prone, so we are going to use a utility library called **formidable**. [Here's the GitHub repo](https://github.com/felixge/node-formidable), it has over 4000 stars and is well-maintained.

You can install it using:

```bash
npm install formidable
```

Then include it in your Node.js file:

```js
const express = require('express')
const app = express()
const formidable = require('formidable')
```

Now, in the `POST` endpoint on the `/submit-form` route, we instantiate a new Formidable form using `formidable.IncomingForm()`:

```js
app.post('/submit-form', (req, res) => {
  new formidable.IncomingForm()
})
```

After doing so, we need to be able to parse the form. We can do so synchronously by providing a callback, which means all files are processed, and once formidable is done, it makes them available:

```js
app.post('/submit-form', (req, res) => {
  new formidable.IncomingForm().parse(req, (err, fields, files) => {
    if (err) {
      console.error('Error', err)
      throw err
    }
    console.log('Fields', fields)
    console.log('Files', files)
    files.map(file => {
      console.log(file)
    })
  })
})
```

Or, you can use events instead of a callback. For example, to be notified when each file is parsed, or other events such as completion of file processing, receiving a non-file field, or if an error occurred:

```js
app.post('/submit-form', (req, res) => {
  let form = new formidable.IncomingForm();
  
  // you can change the upload directory here like this
  form.uploadDir = '/uploads/'; 
  
  form
    .on('field', (name, field) => {
      console.log('Field', name, field)
    })
    .on('fileBegin', (name, file) => {
      // it's also possible to set the upload directory for individual files here like:
      file.path = '/otherUploads/';
    }
    .on('file', (name, file) => {
      console.log('Uploaded file', name, file)
    })
    .on('aborted', () => {
      console.error('Request aborted by the user')
    })
    .on('error', (err) => {
      console.error('Error', err)
      throw err
    })
    .on('end', () => {
      res.end()
    })
    
    form.parse(req);
})
```

Whichever way you choose, you'll get one or more Formidable.File objects, which give you information about the file uploaded. These are some of the methods you can call:

- `file.size`, the file size in bytes
- `file.path`, the path the file is written to
- `file.name`, the name of the file
- `file.type`, the MIME type of the file

The path defaults to the temporary folder and can be modified if you listen for the `fileBegin` event:

```js
app.post('/submit-form', (req, res) => {
  new formidable.IncomingForm().parse(req)
    .on('fileBegin', (name, file) => {
        file.path = __dirname + '/uploads/' + file.name
    })
    .on('file', (name, file) => {
      console.log('Uploaded file', name, file)
    })
    //...
})
```
