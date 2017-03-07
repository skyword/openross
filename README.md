# The OpenRoss image processing service

The OpenRoss image service provides a way of serving dynamically resized images from Amazon S3 in a way that is fast, efficient, and auto-scales with traffic.
We have a blog post describing this service in more detail at http://developers.lyst.com/data/images/2014/06/23/openross/.

## Motivation
At Lyst, we scrape, and have scraped, millions of products that all have at least one image.
In our infancy, we saved any product image into 10 preset sizes, and then chose the nearest one to our needs.
As we grew, this solution was no longer optimum for the levels of traffic we were experiencing and nor was it appropriate for our mobile app.

To address this, we created our BobRoss imaging service which generates a new size on the fly when we need it.
These images are then cached in CloudFront, effectively meaning that Bobross only paints his subjects once, but with more flexibility on dimensions and, in the future, effects.

Since we rolled BobRoss out into production on an auto-scaling Amazon cluster, we have decreased page load times and lowered our bandwidth usage by ensuring we only serve images of the exact size.

As this was so useful for us, we have decided to open source our solution as OpenRoss.

## Requirements

* Python 2.7
* Twisted 13.1
* GraphicsMagick 1.3.19
* nginx 1.6.0

## Installation for testing/debug

1. Clone openross.
2. Setup a virtual environment (optional).
3. Run `pip install -e .`
4. Change directory into the inner `openross` directory.
5. Add your AWS credentials and directory paths to `~/settings.py`
```python
AWS_ACCESS_KEY_ID = "MYKEYID"
AWS_SECRET_ACCESS_KEY = "MYSECRETKEY"
IMAGES_STORE = 'MYBUCKET'
CACHE_LOCATION = '/<path>/<to>/<cache>/'  # e.g. '/srv/http/cache/'
WEB_CACHE_LOCATION = '/url/to/endpoint'  # e.g. '/'
DEBUG = True
```
6. Run `twisted -n openross`.
7. Configure nginx using the `nginx.conf-snippet` file for help.
8. Start nginx.
9. Navigate to `http://localhost/path/to/image/in/your/s3/bucket`.

## Usage

OpenRoss uses a very simple URL scheme http://host/WIDTH/HEIGHT/MODE/path/to/image

Width/height pairs can be white listed in the settings file, or the white list can be turned off for internal development use.

OpenRoss comes bundled with 3 modes: crop, resize, and resizecomp.  Now there's 4!
* Crop does what you think it does, and crops an image into a given size. (No, actually it really doesn't.  It crops it based on the center of the image.  You do not get to pick x1, x2, y1, y2, etc. See Fullcrop below).
* Resize does a simple box resize.
* ResizeComp does a resize, followed by compostiting the image onto a white background---which ensures the image is always an expected size.
* Fullcrop does an actual crop based on user selected values.  Full crop uses http://host/X1/Y1/WIDTH/HEIGHT/fullcrop/path/to/image

New modes can be added easily in `image_modes.py` and blurring, colourising, desaturation are good starting points if you require them.

### Timings and error reporting

OpenRoss supports sending pipeline timings to statsd and errors to sentry.
Your stats server location and sentry_dsn can be set in your settings file.

## FAQS

**What if I find a bug?**

If you find a bug, please make an issue or a pull request with steps on how to reproduce, screenshots, and (if possible) a codefix.
We will be active in monitoring this project.

**What performance characteristics does OpenRoss have?**

We have a blog post talking about performance in more depth available at http://developers.lyst.com/data/images/2014/06/23/openross/ where we go into detail, however at a rate of 110-120 req/s we have a mean responce time of 200ms.
Bear in mind, that we use CloudFront, so 110-120 req/s are only requests for new images that are not in the cache or have recently expired.

**What happens if I change how a mode works, but use CloudFront?**

If you have set a sensible expiry in your cache (we use a week) the changes will go into effect for new images immediately, and will take hold in old images as they are expunged from the cache.
If you have set a larger expirey time for your cache, then you will have to individually remove them from Cloudfront, which is a tedius and timeconsuming process.

**My cache is full!?**

As the cache is in local storage it can fill up quite fast.
There are many solutions to clearing it, the option that we went for was a cron job that runs every 5 minutes.
```bash
find /path/to/cache -ignore_readdir_race -mindepth 3 -type f -name \"*jpeg\" -mmin +30 -delete >/dev/null 2>&1
```
This removes all images that have written more than 30 minutes ago.

**What if I don't store my images in S3?**

If you dont use Amazon S3 to store your images, but still want to use OpenRoss, fear not!

Modify your `~/.openross.py' to turn off the S3 Downloader pipeline and set the internal cache location to your media director:
```python
AWS_ACCESS_KEY_ID = "MYKEYID"
AWS_SECRET_ACCESS_KEY = "MYSECRETKEY"
IMAGES_STORE = 'MYBUCKET'
DEBUG = True         

# Local media

IMAGE_PIPELINES = [
    'pipeline.cache_check.CacheCheck',
    'pipeline.resizer.Resizer',
    'pipeline.cacher.Cacher',
]
CACHE_LOCATION = '/<path>/<to>/<media>/'
```

Next is to modify your nginx root to be where you keep your images, so in our configuation snippet change `root /srv/http/cache;` to `root /<path>/<to>/<media>/;`

This essentially uses your media directory as OpenRoss' cache, so it never has to go to S3 and works as normal.
