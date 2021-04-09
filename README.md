# site
Site through actions, cloudformation.

A given page consists of:

* Template files
* A list of content files that the above Python function needs as input
* A Python function that produces data for that template

Examples:

Music.html
* Depends on templates music.html.templ, head.html.templ, and foot.html.templ.
* Depends on songs.txt
* Python method reads songs.txt, returns a dict that is passed to Jinja renderer.

```
@template_depends('music.html.templ', 'head.html.templ', 'foot.html.templ')
@content_depends('songs.txt)
def make_music():
    pass
```

Gallery Homepage (gallery_name/index.html)
* Depends on gallery.html.templ and head/foot.
* Depends on all photos in the gallery
* A Python function reads cached info about all the photos in the gallery and produces a dict that is passed to the renderer.

```
@template_depends('gallery.html.tmpl', ...)
@content_depends('galleries/photostream/thumbs/*jpg')
```

Individual gallery pages (gallery_name/<name>.html)
* Depends on galpage.html.templ and head/foot
* Depends on a given photo in the gallery (its own) AND the photo before and after it in chronological capture order
* A Python function reads the cached information about this photo and the ones before and after it. It regenerates itself from this information.

```
@content_depends(???)
```

Properties of an OutFile:
* output location
* template to render
* data to pass to template

WHAT GENERATES THUMBNAILS
* lambda makes the most sense - any jpg change to large photos can trigger update to thumb and metadata DB

WHAT GENERATES THE SITE
* lambda, hard to debug, deploy etc
* github actions, easy to program and debug, can read from metadata DB easily, easy to run on a laptop as well

WHAT TRIGGERS IT
* cron, exensive
* changes, implies lambda at least to kick off github action

STEP 1: Thumbnail generation lambda that puts metadata in Dynamo
STEP 2: Script that generates at least 1 page running in github
STEP 3: Script that generates galleries running in github
STEP 4: Script that generates entire site running in github
STEP 5: Lambda triggering github on new photo upload

For site generation, there are some annoyances.

First, while regenerating the entire site takes only a few seconds if the image data is available from Dynamo (which we guarantee it is), uploading it all to S3 is expensive, and "s3 sync" will always upload newer versions, so it's somewhat time-consuming and expensive to pay for all those S3 operations.

In our current world, we avoid needing to re-upload everything by simply not regenerating things that haven't changed. We do this two ways: First, using the Make/DoIt file dependency functionality to avoid regenerating things whose dependencies haven't changed, but second a klooge that reads a file from disk and doesnt write it back if it hasn't changed.

That latter strategy is what currently keeps us from regenerating all the gallery HTML pages every time any image changes. We cannot know ahead of time, for a given image, whether it's forward/back navigation needs to change. So instead we regenerate everything and only write new files if necessary.

If the canonical site lives remotely instead of on our laptop, then that method goes away: if we are going to read from S3 anyway, we might as well just rewrite everything.

Content lives in a private repository and is sync'd to an S3 bucket, A.
Code lives in a public repository and is sync'd to 