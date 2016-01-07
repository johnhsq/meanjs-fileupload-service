Add [ng-file-upload](https://github.com/danialfarid/ng-file-upload) to [MEAN.JS](http://meanjs.org/)
===================
Using MEAN.JS and ng-file-upload to enable image upload to the Article module.

**Notes**: [MEAN.JS 0.4.2 (2015-11-11)](https://github.com/meanjs/mean/releases/tag/v0.4.2) [ng-file-upload 11.0.0 (2015-12-25)](https://github.com/danialfarid/ng-file-upload/releases/tag/11.0.0)


## Create File Upload Web Service
I created REST API to upload files first and it will be used by Articles to upload images.

#### Install Packages
* Server side, I use connect-multiparty. Multer is already included in MEAN.JS

```bash
$ npm install connect-multiparty --save
```

#### Change configuration
* /config/env/default.js

Configure the file upload path. Make sure you create the folder first.
```js
fileUpload: {
  dest: './modules/core/client/img/uploads/', // file upload destination path
  limits: {
    fileSize: 10*1024*1024 // Max file size in bytes (10 MB)
  }
}
```
* /config/lib/multer.js

Create file upload filter
```js
module.exports.imageUploadFileFilter = function (req, file, cb) {
  if (file.mimetype !== 'image/png' && file.mimetype !== 'image/jpg' && file.mimetype !== 'image/jpeg' && file.mimetype !== 'image/gif') {
    return cb(new Error('Only image files are allowed!'), false);
  }
  cb(null, true);
};
```

#### Add Server Routes
* /modules/core/server/routes/core.server.routes.js

Create Server Router, which will be the REST API endpoint.
```js
app.route('/api/uploads')
  .post(multipartyMiddleware, core.uploads);
```

#### Add Server Controller
* /modules/core/server/controllers/core.server.controller.js

Create Server Controller.
```js
exports.uploads = function (req, res) {
  // console.log('req.headers.content-type:'+req.headers['content-type']); //must be multipart/form-data
  // console.log('req.files.uploadedFile: '+req.files.uploadedFile); //file object
  // console.log('req.files.uploadedFile.fieldname: '+req.files.uploadedFile.fieldName);// "uploadedFile"
  // console.log('req.files.uploadedFile.name: '+req.files.uploadedFile.name); // original filename
  // console.log('req.files.uploadedFile.originalFilename: '+req.files.uploadedFile.originalFilename); // orininal Filename
  // console.log('req.files.uploadedFile.type: '+req.files.uploadedFile.type); //image/jpeg
  // console.log('req.files.uploadedFile.size: '+req.files.uploadedFile.size); // file size
  // console.log('req.body.testkey: '+req.body.testkey); // submited form

  var file=req.files.uploadedFile;
  var user = req.user;
  var message = null;

  //the target folder is ./modules/core/client/img/uploads/base64(username)/
  var userEncode = new Buffer(user.username).toString('base64');
  var destFolder = path.join(path.resolve('./'),config.uploads.fileUpload.dest,userEncode);
  var newFilename = Date.now()+"-"+file.originalFilename;
  var destFile = destFolder+"/"+newFilename;
  //var destURL = req.protocol + '://' + req.get('host') + config.uploads.fileUpload.dest+userEncode+"/"+Date.now()+".jpg";
  var destURL = config.uploads.fileUpload.dest+userEncode+"/"+newFilename;

  //config multer, somehow the diskStorage() is not working
  var storage = multer.diskStorage({
    destination: function (req, file, cb) {
      cb(null, destFolder);
    },
    filename: function (req, file, cb) {
      cb(null, file.originalname + '-' + Date.now());
    }
  });

  var upload = multer({ storage : storage}).single('uploadedFile');
  // Filtering to upload only images
  upload.fileFilter = require(path.resolve('./config/lib/multer')).imageUploadFileFilter;

  // upload file
  upload(req,res,function (err) {
    if(err) {
      return res.status(400).send({
        message: 'Error occurred while uploading profile picture'
      });
    }else{
      // For some reason, the diskStorage function of Multer doesn't work.
      // The following code is to move the file to the destination folder.
      var stat =null;
      try {
        stat = fs.statSync(destFolder);
      } catch (err) {
        fs.mkdirSync(destFolder);
      }
      if (stat && !stat.isDirectory()) {
        throw new Error('Directory cannot be created because an inode of a different type exists at "' + destFolder + '"');
      } else {
        fs.rename(file.path, destFile, function(err) {
          if (err) throw err;
          // delete the temporary file, so that the explicitly set temporary upload dir does not get filled with unwanted files
          fs.unlink(file.path, function() {
            if (err) {
                throw err;
            }else{
              return res.status(200).send({
                uploadedURL: destURL,
                uploadedFile: destFile,
                file: JSON.stringify(req.files),
                message: 'File is uploaded to ' + destURL
              });
            }
          });
        });
      }
    }
  });
};
```
At current point, you should be able to use the REST API to upload file. You can try postman in Chrome like the images below. Please make sure you set Key as "uploadedFile".

<img src="https://raw.githubusercontent.com/johnhsq/meanjs-fileupload-service/master/img/Picture1.png" width="405">
<img src="https://raw.githubusercontent.com/johnhsq/meanjs-fileupload-service/master/img/Picture2.png" width="405">

## Add Image Feature to Article
I use ng-file-upload for the client side file upload.

#### Install Packages
* Install ng-file-upload

```bash
$ bower install ng-file-upload --save
```
#### Change configuration
* /config/assets/default.js

Add ng-file-upload modules
```js
'public/lib/ng-file-upload/ng-file-upload-shim.js',
'public/lib/ng-file-upload/ng-file-upload.js'
```
* /config/assets/production.js

Add minified version to production configuration
```js
'public/lib/ng-file-upload/ng-file-upload-shim.min.js',
'public/lib/ng-file-upload/ng-file-upload.min.js'
```
* /modules/core/client/app/config.js

Inject ngFileUpload module to dependencies
```js
  var applicationModuleVendorDependencies = ['ngResource', 'ngAnimate', 'ngMessages', 'ui.router', 'ui.bootstrap', 'ui.utils', 'angularFileUpload', 'ngFileUpload'];
```
#### Add image URL to Article Server Models
* /modules/articles/server/models/article.server.model.js

Add image URL field to the Article Model
```js
articleImageURL:{
  type: String,
  default: '',
  trim: true
},
```
#### Change Article Views
* /modules/articles/client/views/create-article.client.view.html

Add Featured Image to the form
```html
<div class="form-group" show-errors>
  <label for="title">Featured Image </label>
  <button type="button" class="btn btn-secondary" ng-show="uploadedFile == null" type="file"
    ngf-select="uploadFiles($file, $invalidFiles)"
    ng-model="uploadedFile" name="file" ngf-model-invalid="errorFiles"
    accept="image/*" ngf-max-size="1MB">
    Select Featured Image</button>
  <div class="text-center">
    <img ng-show="articleForm.file.$valid" ngf-thumbnail="uploadedFile" class="thumb">
  </div>
  <div class="text-center">
    <div class="progress" ng-show="uploadedFile.progress >= 0">
              <div style="width:{{uploadedFile.progress}}%"
                  ng-bind="uploadedFile.progress + '%'"></div>
    </div>
  </div>
  <div class="text-center">
    <button type="button" class="btn btn-secondary" ng-click="uploadedFile = null" ng-show="uploadedFile">Remove</button>
  </div>
  <div class="alert alert-danger" ng-show="articleForm.file.$error.maxSize">File too large
               {{errorFiles[0].size / 1000000|number:1}}MB: max 1M</div>
</div>
```
* /modules/articles/client/views/edit-article.client.view.html

Add Featured Image to the form
```html
<div class="form-group" show-errors>
  <label for="title">Featured Image </label>
  <button ng-show="article.articleImageURL==null || article.articleImageURL==''" type="button" class="btn btn-secondary" type="file"
    ngf-select="uploadFiles($file, $invalidFiles)"
    ng-model="article.articleImageURL" name="file" ngf-model-invalid="errorFiles"
    accept="image/*" ngf-max-size="1MB">
    Select Featured Image</button>
  <div class="text-center" ng-show="article.articleImageURL" >
    <img ng-src="{{article.articleImageURL}}" ngf-thumbnail="article.articleImageURL" class="thumb"/>
  </div>
  <div class="text-center" ng-show="article.articleImageURL">
    <button type="button" class="btn btn-secondary" ng-click="article.articleImageURL = null" >Remove</button>
  </div>
  <div class="alert alert-danger" ng-show="articleForm.file.$error.maxSize">File too large
               {{errorFiles[0].size / 1000000|number:1}}MB: max 1M</div>
</div>
```
* /modules/articles/client/views/list-articles.client.view.html

Show Featured Image in the view
```html
<img ng-src="{{article.articleImageURL}}" height="100" />
```
* /modules/articles/client/views/view-article.client.view.html

Show Featured Image in the view
```html
<img ng-src="{{article.articleImageURL}}" height="100" />
```
#### Change Article Client Controller
* /modules/articles/client/controllers/articles.client.controller.js

Inject Upload and $timeout services to the Article Controller
```js
$scope.uploadFiles = function(file, errFiles) {
    $scope.uploadedFile = file;
    $scope.errFile = errFiles && errFiles[0];
    if (file) {
        file.upload = Upload.upload({
            url: '/api/uploads',
            data: {uploadedFile: file}
        });

        file.upload.then(function (response) {
            console.log('File is successfully uploaded to ' + response.data.uploadedURL);
            $scope.articleImageURL = response.data.uploadedURL;
            $timeout(function () {
                file.result = response.data;
            });
        }, function (response) {
            if (response.status > 0)
                $scope.errorMsg = response.status + ': ' + response.data;
        }, function (evt) {
            file.progress = Math.min(100, parseInt(100.0 *
                                     evt.loaded / evt.total));
        });
    }
};


// Create new Article
$scope.create = function (isValid) {
  $scope.error = null;

  if (!isValid) {
    $scope.$broadcast('show-errors-check-validity', 'articleForm');

    return false;
  }

  // Create new Article object
  var article = new Articles({
    title: $scope.title,
    content: $scope.content,
    articleImageURL: $scope.articleImageURL
  });

  // Redirect after save
  article.$save(function (response) {
    $location.path('articles/' + response._id);

    // Clear form fields
    $scope.title = '';
    $scope.content = '';
    $scope.articleImageURL = '';
  }, function (errorResponse) {
    $scope.error = errorResponse.data.message;
  });
};

// Update existing Article
$scope.update = function (isValid) {
  $scope.error = null;

  if (!isValid) {
    $scope.$broadcast('show-errors-check-validity', 'articleForm');

    return false;
  }

  var article = $scope.article;
  article.articleImageURL = $scope.articleImageURL;

  article.$update(function () {
    $location.path('articles/' + article._id);
  }, function (errorResponse) {
    $scope.error = errorResponse.data.message;
  });
};
```
