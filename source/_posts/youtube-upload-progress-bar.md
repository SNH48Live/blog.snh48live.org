---
title: Command line YouTube upload with progress bar
date: 2017-07-16 00:37:00
tags:
---
The first technical post.

I upload videos to the channel using an adapted version of [`upload_video.py`](https://github.com/youtube/api-samples/blob/master/python/upload_video.py) from YouTube's API code samples. The script does not provide a progress bar. This is usually not a problem because my connection is pretty fast, and when I upload from my Mac I can always bring up Activity Monitor to check bytes uploaded by the python process. However, when I upload from a shared remote server, sometimes due to spikes in load not under my control, upload would seem slower than usual; so I do want to see a progress bar occasonally.

`upload_video.py` uses [`MediaFileUpload`](https://google.github.io/google-api-python-client/docs/epy/googleapiclient.http.MediaFileUpload-class.html) which is a blackbox taking only the file path, and when you call [`next_chunk`](https://google.github.io/google-api-python-client/docs/epy/googleapiclient.http.HttpRequest-class.html#next_chunk) on an insert request, technically a [`MediaUploadProgress`](https://google.github.io/google-api-python-client/docs/epy/googleapiclient.http.MediaUploadProgress-class.html) is returned which tells you the percentage completed, but realistically if you're streaming the file (you should), the whole file is treated as a single "chunk" (`next_chunk` is only called once), so it's pretty much useless.

I looked through the docs and, fortunately, there is a lower level [`MediaIoBaseUpload`](https://google.github.io/google-api-python-client/docs/epy/googleapiclient.http.MediaIoBaseUpload-class.html) class that is less of a blackbox. With this class we can simply hook progress bar callbacks into `read` and `seek` of an `io.IOBase` object. Here's some skeleton code (where I use an `io.BufferedReader` wrapped around an `io.FileIO` object, just like what is done by [`open`](https://github.com/python/cpython/blob/v3.6.1/Lib/_pyio.py#L40-L249)):

```py
import io
import mimetypes
import os


class FileReaderWithProgressBar(io.BufferedReader):

    def __init__(self, file):
        raw = io.FileIO(file)
        buffering = io.DEFAULT_BUFFER_SIZE
        try:
            block_size = os.fstat(raw.fileno()).st_blksize
        except (OSError, AttributeError):
            pass
        else:
            if block_size > 1:
                buffering = block_size
        super().__init__(raw, buffering)

    def seek(self, pos, whence=0):
        abspos = super().seek(pos, whence)
        # TODO: report position: abspos
        return abspos

    def read(self, size=-1):
        result = super().read(size)
        # TODO: report position: self.tell()
        return result


class MediaFileUploadWithProgressBar(googleapiclient.http.MediaIoBaseUpload):

    def __init__(self, file, mimetype=None, chunksize=googleapiclient.http.DEFAULT_CHUNK_SIZE,
                 resumable=False):
        fp = FileReaderWithProgressBar(file)
        if mimetype is None:
            mimetype, _ = mimetypes.guess_type(file)
        super().__init__(fp, mimetype, chunksize=chunksize, resumable=resumable)
```

Any progress bar class can be hooked into `FileReaderWithProgressBar`, with the update method called at places marked as `TODO`.

See my [`upload`](https://github.com/SNH48Live/SNH48Live/blob/master/bin/upload) script for a complete implementation (commit [`111c200`](https://github.com/SNH48Live/SNH48Live/blob/111c2002ff314e1566baedec90430ef2025bbded/bin/upload#L86) in case the file is moved or refactored in the future).
