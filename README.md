# Pixlserv

A Go server for on-the-fly processing and serving of images. A self-hosted image management solution for websites and mobile apps.

1. Get the code and install the server

2. Connect it to Amazon S3 or use local storage, connect it to redis

3. Start uploading and transforming images

This is version 0.1. All feedback welcome: hello _at_ reshnesh _dot_ com


## Installation

First get pixlserv's source code by either cloning this repository to your Go workspace:

```
git clone https://github.com/ReshNesh/pixlserv
```

or by:

```
go get github.com/ReshNesh/pixlserv
```

Navigate to the project directory and fetch dependencies:

```
go get
```

Now, build the project:

```
go build
```


## Usage

Images are requested from the server by accessing a URL of the following format: `http://server/image/parameters/filename`. Parameters are strings like `transformation_value` connected with commas, e.g. `w_400,h_300`. A full URL could look like this: `http://pixlserv.com/image/w_400,h_300/logo.jpg`. Once an image is transformed in some way the copy is cached which means it can be accessed quickly next time.

Upload is done by sending an image file as an `image` field of a POST request to `http://server/upload`.

Authorisation can be easily set up to require an API key between `server` and `image` (or `upload`) in the example URLs above.

### Using pixlserv locally

Start redis (see the [Requirements](#requirements) section for details). Create a directory `local-images` with some JPEG or PNG images in the same directory where you installed pixlserv. Then run:

```
./pixlserv run config/example.yaml
```

This will run the server using a simple configuration defined in [config/example.yaml](config/example.yaml). You are encouraged to look at the Configuration section below, create a copy of the sample configuration file and customise it to suit your needs.

Assuming you copied a file `cat.jpg` to the `local-images` directory you can now access [http://localhost:3000/image/t_square/cat.jpg](http://localhost:3000/image/t_square/cat.jpg) using your browser.

### Using pixlserv with Heroku

Heroku is a popular platform-as-a-service (PaaS) provider so we will have a look at a more detailed description of how to make pixlserv work on Heroku's infrastructure.

As Heroku uses an [ephemeral filesystem](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem) which doesn't allow persistent storage of data you will need to use Amazon S3. Please make sure you have a bucket and a user authorised to access the bucket using the instructions in the [Amazon S3](#amazon-s3) section below.

Clone this repository somewhere on your disk. If you do not plan to run the server locally it doesn't need to be in Go's workspace.

Navigate to the project directory and switch to the `heroku` branch:

```
git checkout heroku
```

[Create a Heroku app](https://devcenter.heroku.com/articles/heroku-command) for the new repository. You will also need a redis add-on such as [Redis Cloud](https://addons.heroku.com/rediscloud).

Now, you will need to configure a few environment variables, replace `???` with your S3 credentials:

```
heroku config:set BUILDPACK_URL=https://github.com/kr/heroku-buildpack-go.git
heroku config:set AWS_ACCESS_KEY_ID=??? AWS_SECRET_ACCESS_KEY=??? PIXLSERV_S3_BUCKET=???
```

Find out the URL of your redis instance, if you're using Redis Cloud it is:

```
heroku config:get REDISCLOUD_URL
```

Now, use the URL to set up a new environment variable:

```
heroku config:set PIXLSERV_REDIS_URL=???
```

Either create a new pixlserv configuration file (more info below) or use the example one for now and then make sure `web: bin/pixlserv run CONFIG_FILE` (where `CONFIG_FILE` is the path to your file) is the contents of a new file called `Procfile`:

```
echo web: bin/pixlserv run config/example.yaml > Procfile
```

Add newly created files to Git (`Procfile` + your new configuration file) and commit.

If there are no Heroku dynos running start one up (if you just created the Heroku app it will be the case):

```
heroku ps:scale web=1
```

Push the local `heroku` branch to heroku's git repository `master` branch:

```
git push heroku heroku:master
```

Once, the application is deployed open the browser with the index page to see if everything is fine:

```
heroku open
```

You should now see the text: It works!

You can now start requesting images stored in your S3 bucket.

Please note: uploads will not work if you're using the example configuration file without creating an API key first (refer to the [Authentication](#authentication) section to learn more about authentication). You will want to generate a key:

```
heroku run bin/pixlserv api-key add
```

Then restart the server and use the key as part of the URL you POST to (`http://APPNAME.herokuapp.com/KEY/upload`).


## Configuration

Pixlserv supports 2 types of underlying storage: local file system and Amazon S3. If environment variables `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `PIXLSERV_S3_BUCKET` are detected the server will try to connect to S3 given the given credentials. Otherwise, local storage will be used. The path at which images will be stored locally can be specified using the `local-path` configuration option.

[//]: # (TODO: more info)
Other configuration options include `throttling-rate`, `allow-custom-transformations`, `allow-custom-scale`, `async-uploads`, `authorisation`, `cache`, `jpeg-quality`, `transformations` and `upload-max-file-size`. See [config/example.yaml](config/example.yaml) for an example.

Configuration is kept in a [YAML](http://en.wikipedia.org/wiki/YAML) file. In some cases its syntax could be confusing if you haven't used YAML before so please refer to some online documentation. For example, hexadecimal colours need to be in quotes (as hash would start a comment otherwise). A string `n` specifying gravity could be interpreted as a shorthand for boolean `No` and so needs to be put in quotes too.

### Amazon S3

To use Amazon S3 as your storage create a bucket and a user with access to the bucket and at least the following permissions: `s3:GetObject`, `s3:DeleteObject`, `s3:PutObject` and `s3:ListBucket`. Make sure to set up the environment variables mentioned above to make the server connect to S3 instead of using local storage.

The policy for an S3 user should contain something like this (where `<BUCKET>` is the name of your bucket):

```
"Effect": "Allow",
"Action": [
    "s3:DeleteObject",
    "s3:GetObject",
    "s3:ListBucket",
    "s3:PutObject"
],
"Resource": [
    "arn:aws:s3:::<BUCKET>/*"
]
```


## Supported transformations

### Resizing

| Parameter | Meaning                       |
| --------- | ----------------------------- |
| h_X       | sets height of the image to X |
| w_X       | sets width of the image to X  |


### Cropping

| Parameter | Meaning                                                                                                       |
| --------- | ------------------------------------------------------------------------------------------------------------- |
| c_e       | exact, image scaled exactly to given dimensions (default)                                                     |
| c_a       | all, the whole image will be visible in a frame of given dimensions, retains proportions                      |
| c_p       | part, part of the image will be visible in a frame of given dimensions, retains proportions, optional gravity |
| c_k       | keep scale, original scale of the image preserved, optional gravity                                           |


### Gravity

For some cropping modes gravity determines which part of the image will be shown.

| Parameter | Meaning                         |
| --------- | ------------------------------- |
| g_n       | north, top edge                 |
| g_ne      | north east, top-right corner    |
| g_e       | east, right edge                |
| g_se      | south east, bottom-right corner |
| g_s       | south, bottom edge              |
| g_sw      | south west, bottom-left corner  |
| g_w       | west, left edge                 |
| g_nw      | north west, top-left corner     |
| g_c       | center                          |


### Filter/colouring

| Parameter   | Meaning   |
| ----------- | --------- |
| f_grayscale | grayscale |


### Scaling (retina)

Scales the image up to support retina devices. For example to generate a thumbnail of an image (`image.jpg`) at twice the size request `image@2x.jpg`. Only positive integers are accepted as valid scaling factors.


### Named transformations

In your configuration file you can specify transformations using parameters described above and then give each transformation a name. The transformation can then be invoked using a `t_mytransformation` URL parameter.

Named transformations can also be set to be `eager`. Such transformations will be run for all images uploaded using the server straight after the upload happens.

Another feature of named transformations are watermarks. Images can be automatically watermarked using an image stored along with all the other images locally or in S3. Please refer to the example config file for details. Note: if you supply scaled up watermarks (`watermark@2x.png`) these will be used for scaled images.


## Authentication

The server can be set up to require an API key to be passed as part of the URL when requesting or uploading an image. This is done in the `authorisation` section of a configuration file.

API keys can be added, removed and modified by running `./pixlserv api-key COMMAND`. Run this without `COMMAND` to see all the available commands. Once API keys are modified, the server needs to be restarted to use the new settings.


## Requirements

A running [redis](http://redis.io/) instance is required for the server to be able to maintain a cache of images. Check the redis website to find out how to download and install redis. If you run redis on a different port than the default 6379 please make sure to set up a `PIXLSERV_REDIS_PORT` environment variable with the port you are using.


## Future development

* a Javascript library for easier uploads from the browser
* Django, Rails support for generating image URLs on the server
* GUI to manage images in the browser
* a sample application
