#### Using Google Cloud Storage as your storage provider
This example shows how to add and retrieve files using Google Cloud Storage.
Additionally, this example will also show you how to list uploaded files and remove any of them.

*See production-ready code below.*

##### Step 1: install [gcloud-node](https://github.com/googlecloudplatform/gcloud-node)

```shell
npm install --save gcloud
```
Or
```shell
meteor npm install gcloud
```

##### Step 2: Setup your Google Cloud Storage
- Sign into the Google Cloud Console site: https://console.developers.google.com
- Go to your project and under Storage click on Create Bucket, and name your Bucket
  * Don't forget the name of this bucket, you'll need it later
- Under APIs & Auth click on Credentials
- Create an OAuth Service Account for your project (*if you don't already have one*)
- If you do not have a private key, click "*Generate new key*"
  * This will download a JSON file to your computer
  * Keep this JSON file, it is your key to the account, and you will need it later
  * Also, __GUARD THIS JSON FILE AS YOUR LIFE__. IF ANYONE GETS AHOLD OF IT, THEY WILL HAVE FULL ACCESS TO YOUR ACCOUNT!

```javascript
var gcloud, gcs, bucket, bucketMetadata, Request, bound, Collections = {};

if (Meteor.isServer) {
  gcloud = Npm.require('gcloud')({
    projectId: 'YOUR_PROJECT_ID', // <-- Replace this with your project ID
    keyFilename: 'YOUR_KEY_JSON'  // <-- Replace this with the path to your key.json
  });
  gcs = gcloud.storage();
  bucket = gcs.bucket('YOUR_BUCKET_NAME'); // <-- Replace this with your bucket name
  bucket.getMetadata(function(error, metadata, apiResponse){
    if (error) {
      console.error(error);
    } else {
      bucketMetadata = metadata;
    }
  });
  Request = Npm.require('request');
  bound = Meteor.bindEnvironment(function(callback){
    return callback();
  });
}

Collections.files = new FilesCollection({
  debug: false, // Set to true to enable debugging messages
  throttle: false,
  storagePath: 'assets/app/uploads/uploadedFiles',
  collectionName: 'uploadedFiles',
  allowClientCode: false,
  onAfterUpload: function(fileRef) {
    // In the onAfterUpload callback, we will move the file to Google Cloud Storage
    var self = this;
    _.each(fileRef.versions, function(vRef, version){
      // We use Random.id() instead of real file's _id
      // to secure files from reverse engineering
      // As after viewing this code it will be easy
      // to get access to unlisted and protected files
      var filePath = "files/" + (Random.id()) + "-" + version + "." + fileRef.extension;
      // Here we set the neccesary options to upload the file, for more options, see
      // https://googlecloudplatform.github.io/gcloud-node/#/docs/v0.36.0/storage/bucket?method=upload
      var options = {
        destination: filePath,
        resumable: true,
        public: true
      };

      bucket.upload(fileRef.path, options, function(error, file){
        bound(function(){
          var upd;
          if (error) {
            console.error(error);
          } else {
            upd = {
              $set: {}
            };
            upd['$set']["versions." + version + ".meta.pipeFrom"] = bucketMetadata.selfLink + '/' + filePath;
            upd['$set']["versions." + version + ".meta.pipePath"] = filePath;
            self.collection.update({
              _id: fileRef._id
            }, upd, function (error) {
              if (error) {
                console.error(error);
              } else {
                // Unlink original files from FS
                // after successful upload to Google Cloud Storage
                self.unlink(self.collection.findOne(fileRef._id), version);
              }
            });
          }
        });
      });
    });
  },
  interceptDownload: function(http, fileRef, version) {
    var path, ref, ref1, ref2;
    path = (ref= fileRef.versions) != null ? (ref1 = ref[version]) != null ? (ref2 = ref1.meta) != null ? ref2.pipeFrom : void 0 : void 0 : void 0;
    if (path) {
      // If file is moved to Google Cloud Storage
      // We will pipe request to Google Cloud Storage
      // So, original link will stay always secure
      Request({
        url: path,
        headers: _.pick(http.request.headers, 'range', 'accept-language', 'accept', 'cache-control', 'pragma', 'connection', 'upgrade-insecure-requests', 'user-agent')
      }).pipe(http.response);
      return true;
    } else {
      // While the file has not been uploaded to Google Cloud Storage, we will serve it from the filesystem
      return false;
    }
  }
});

if (Meteor.isServer) {
  // Intercept file's collection remove method to remove file from Google Cloud Storage
  var _origRemove = Collections.files.remove;

  Collections.files.remove = function(search) {
    var cursor = this.collection.find(search);
    cursor.forEach(function(fileRef) {
      _.each(fileRef.versions, function(vRef) {
        var ref;
        if (vRef != null ? (ref = vRef.meta) != null ? ref.pipePath : void 0 : void 0) {
          bucket.deleteFiles(vRef.meta.pipePath, function(error) {
            bound(function() {
              if (error) {
                console.error(error);
              }
            });
          });
        }
      });
    });
    // Call the original removal method
    _origRemove.call(this, search);
  };
}
```