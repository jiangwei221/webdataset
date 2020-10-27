[![Test](https://github.com/tmbdev/webdataset/workflows/Test/badge.svg)](https://github.com/tmbdev/webdataset/actions?query=workflow%3ATest)
[![DeepSource](https://static.deepsource.io/deepsource-badge-light-mini.svg)](https://deepsource.io/gh/tmbdev/webdataset/?ref=repository-badge)

# WebDataset

WebDataset is a PyTorch Dataset (IterableDataset) implementation providing
efficient access to datasets stored in POSIX tar archives and uses only sequential/streaming
data access. This brings substantial performance advantage in many compute environments, and it
is essential for very large scale training.

WebDataset implements standard PyTorch `IterableDataset` interface and works with the PyTorch `DataLoader`.
Access to datasets is as simple as:

```Python
dataset = wds.WebDataset(url).shuffle(1000).decode("torchrgb").to_tuple("jpg;png", "json")
dataloader = torch.utils.data.DataLoader(dataset, num_workers=4, batch_size=16)

for inputs, outputs in dataloader:
    ...
```

In that code snippet, `url` can refer to a local file, a local HTTP server, a cloud storage object, an object
on an object store, or even the output of arbitrary command pipelines.

WebDataset fulfills a similar function to Tensorflow's TFRecord/tf.Example
classes, but it is much easier to adopt because it does not actually
require any kind of data conversion: data is stored in exactly the same
format inside tar files as it is on disk, and all preprocessing and data
augmentation code remains unchanged.

# Installation and Documentation

    $ pip install webdataset

For the Github version:

    $ pip install git+https://github.com/tmbdev/webdataset.git

Documentation: [ReadTheDocs](http://webdataset.readthedocs.io)

Examples:

- [loading videos](https://github.com/tmbdev/webdataset/blob/master/docs/video-loading-example.ipynb)
- [splitting raw videos into clips for training](https://github.com/tmbdev/webdataset/blob/master/docs/ytsamples-split.ipynb)
- [converting the Falling Things dataset](https://github.com/tmbdev/webdataset/blob/master/docs/falling-things-make-shards.ipynb)

# Introductory Videos

Here are some videos talking about WebDataset and large scale deep learning:

- [Introduction to Large Scale Deep Learning](https://www.youtube.com/watch?v=kNuA2wflygM)
- [Loading Training Data with WebDataset](https://www.youtube.com/watch?v=mTv_ePYeBhs)
- [Creating Datasets in WebDataset Format](https://www.youtube.com/watch?v=v_PacO-3OGQ)
- [Tools for Working with Large Datasets](https://www.youtube.com/watch?v=kIv8zDpRUec)

# Using WebDataset

WebDataset reads dataset that are stored as tar files, with the simple convention that files that belong together and make up a training sample share the same basename. WebDataset can read files from local disk or from any pipe, which allows it to access files using common cloud object stores.


```bash
%%bash
curl -s http://storage.googleapis.com/nvdata-openimages/openimages-train-000000.tar | tar tf - | sed 10q
```

    e39871fd9fd74f55.jpg
    e39871fd9fd74f55.json
    f18b91585c4d3f3e.jpg
    f18b91585c4d3f3e.json
    ede6e66b2fb59aab.jpg
    ede6e66b2fb59aab.json
    ed600d57fcee4f94.jpg
    ed600d57fcee4f94.json
    ff47e649b23f446d.jpg
    ff47e649b23f446d.json



```python
%pylab inline

import torch
from torch.utils.data import IterableDataset
from torchvision import transforms
import webdataset as wds
from itertools import islice

url = "http://storage.googleapis.com/nvdata-openimages/openimages-train-000000.tar"
url = f"pipe:curl -L -s {url} || true"
```

    Populating the interactive namespace from numpy and matplotlib


For starters, let's use the `webdataset.Dataset` class to illustrate how the `webdataset` library works.


```python
dataset = wds.WebDataset(url)

for sample in islice(dataset, 0, 3):
    for key, value in sample.items():
        print(key, repr(value)[:50])
    print()
```

    __key__ 'e39871fd9fd74f55'
    jpg b'\xff\xd8\xff\xe0\x00\x10JFIF\x00\x01\x01\x01\x01
    json b'[{"ImageID": "e39871fd9fd74f55", "Source": "xcli
    
    __key__ 'f18b91585c4d3f3e'
    jpg b'\xff\xd8\xff\xe0\x00\x10JFIF\x00\x01\x01\x00\x00
    json b'[{"ImageID": "f18b91585c4d3f3e", "Source": "acti
    
    __key__ 'ede6e66b2fb59aab'
    jpg b'\xff\xd8\xff\xe0\x00\x10JFIF\x00\x01\x01\x01\x00
    json b'[{"ImageID": "ede6e66b2fb59aab", "Source": "acti
    


There are common processing stages you can add to a dataset to make it a drop-in replacement for any existing dataset. For convenience, common operations are available through a "fluent" interface (as chained method calls).


```python
dataset = (
    wds.WebDataset(url)
    .shuffle(100)
    .decode("rgb")
    .to_tuple("jpg;png", "json")
)

for image, data in islice(dataset, 0, 3):
    print(image.shape, image.dtype, type(data))
```

    (768, 1024, 3) float32 <class 'list'>
    (768, 768, 3) float32 <class 'list'>
    (1024, 1024, 3) float32 <class 'list'>


The `webdataset.Dataset` class has some common operations:

- `shuffle(n)`: shuffle the dataset with a buffer of size `n`; also shuffles shards (see below)
- `decode(decoder, ...)`: automatically decode files (most commonly, you can just specify `"pil"`, `"rgb"`, `"rgb8"`, `"rgbtorch"`, etc.)
- `rename(new="old1;old2", ...)`: rename fields
- `map(f)`: apply `f` to each sample
- `map_dict(key=f, ...)`: apply `f` to its corresponding key
- `map_tuple(f, g, ...)`: apply `f`, `g`, etc. to their corresponding values in the tuple
- `pipe(f)`: `f` should be a function that takes an iterator and returns a new iterator

Stages commonly take a `handler=` argument, which is a function that gets called when there is an exception; you can write whatever function you want, but common functions are:

- `webdataset.ignore_and_stop`
- `webdataset.ignore_and_continue`
- `webdataset.warn_and_stop`
- `webdataset.warn_and_continue`
- `webdataset.reraise_exception`


Here is an example that uses `torchvision` data augmentation the same way you might use it with a `FileDataset`.


```python
def identity(x):
    return x

normalize = transforms.Normalize(
    mean=[0.485, 0.456, 0.406],
    std=[0.229, 0.224, 0.225])

preproc = transforms.Compose([
    transforms.RandomResizedCrop(224),
    transforms.RandomHorizontalFlip(),
    transforms.ToTensor(),
    normalize,
])

dataset = (
    wds.WebDataset(url)
    .shuffle(100)
    .decode("pil")
    .to_tuple("jpg;png", "json")
    .map_tuple(preproc, identity)
)

for image, data in islice(dataset, 0, 3):
    print(image.shape, image.dtype, type(data))
```

    torch.Size([3, 224, 224]) torch.float32 <class 'list'>
    torch.Size([3, 224, 224]) torch.float32 <class 'list'>
    torch.Size([3, 224, 224]) torch.float32 <class 'list'>


You can find the full PyTorch ImageNet sample code converted to WebDataset at [tmbdev/pytorch-imagenet-wds](http://github.com/tmbdev/pytorch-imagenet-wds)

# Sharding and Parallel I/O

In order to be able to shuffle data better and to process and load data in parallel, it is a good idea to shard it; that is, to split up the dataset into several `.tar` files.

WebDataset uses standard UNIX brace notation for sharded dataset. For example, the OpenImages dataset consists of 554 shards, each containing about 1 Gbyte of images. You can open the entire dataset as follows.


```python
url = "http://storage.googleapis.com/nvdata-openimages/openimages-train-{000000..000554}.tar"
url = f"pipe:curl -L -s {url} || true"
dataset = (
    wds.WebDataset(url)
    .shuffle(100)
    .decode("pil")
    .to_tuple("jpg;png", "json")
    .map_tuple(preproc, identity)
)
```

When used with a standard Torch `DataLoader`, this will now perform parallel I/O and preprocessing.


```python
dataloader = torch.utils.data.DataLoader(dataset, num_workers=4, batch_size=16)
images, targets = next(iter(dataloader))
images.shape
```




    torch.Size([16, 3, 224, 224])




```python
torch.Size([16, 3, 224, 224])
```




    torch.Size([16, 3, 224, 224])



The recommended way of using `IterableDataset` with `DataLoader` is to do the batching explicitly in the `Dataset`. In addition, you need to set a nominal length for the `Dataset` in order to avoid warnings from `DataLoader`.


```python
url = "http://storage.googleapis.com/nvdata-openimages/openimages-train-{000000..000554}.tar"
url = f"pipe:curl -L -s {url} || true"
bs = 20

dataset = (
    wds.WebDataset(url, length=int(1e9) // bs)
    .shuffle(100)
    .decode("pil")
    .to_tuple("jpg;png", "json")
    .map_tuple(preproc, identity)
    .batched(20)
)

dataloader = torch.utils.data.DataLoader(dataset, num_workers=4, batch_size=None)
images, targets = next(iter(dataloader))
images.shape
```




    torch.Size([20, 3, 224, 224])



The `ResizedDataset` is also helpful for connecting iterable datasets to `DataLoader`: it lets you set both a nominal and an actual epoch size; it will repeatedly iterate through the entire dataset and return data in chunks with the given epoch size.

# Data Decoding

(to be written)

# Creating a WebDataset

Since WebDatasets are just regular tar files, you can usually create them by just using the `tar` command. All you have to do is to arrange for any files that should be in the same sample to share the same basename. Many datasets already come that way. For those, you can simply create a WebDataset with


```python
$ tar --sort=name -cf dataset.tar dataset/
```

If your dataset has some other directory layout, you can either rearrange the files on disk, or you can use `tar --transform` to get the right kinds of names in your tar file.

You can also create a WebDataset with library functions in this library:

- `webdataset.TarWriter` takes dictionaries containing key value pairs and writes them to disk
- `webdataset.ShardWriter` takes dictionaries containing key value pairs and writes them to disk as a series of shards

Here is a quick way of converting an existing dataset into a WebDataset; this will store all tensors as Python pickles:


```python
sink = wds.TarWriter("dest.tar")
dataset = open_my_dataset()
for index, (input, output) in dataset:
    sink.write({
        "__key__": "sample%06d" % index,
        "input.pyd": input,
        "output.pyd": output,
    })
sink.close()
```

Storing data as Python pickles allows most common Python datatypes to be stored, it is lossless, and the format is fast to decode.
However, it is uncompressed and cannot be read by non-Python programs. It's often better to choose other storage formats, e.g.,
taking advantage of common image compression formats.

If you know that the input is an image and the output is an integer class, you can also write something like this:


```python
sink = wds.TarWriter("dest.tar")
dataset = open_my_dataset()
for index, (input, output) in dataset:
    assert input.ndim == 3 and input.shape[2] == 3
    assert input.dtype = np.float32 and np.amin(input) >= 0 and np.amax(input) <= 1
    assert type(output) == int
    sink.write({
        "__key__": "sample%06d" % index,
        "input.jpg": input,
        "output.cls": output,
    })
sink.close()
```

The `assert` statements in that loop are not necessary, but they document and illustrate the expectations for this
particular dataset. Generally, the ".jpg" encoder can actually encode a wide variety of array types as images. The
".cls" encoder always requires an integer for encoding.

Here is how you can use `TarWriter` for writing a dataset without using an encoder:


```python
sink = wds.TarWriter("dest.tar", encoder=False)
for basename in basenames:
    with open(f"{basename}.png", "rb") as stream):
        image = stream.read()
    cls = lookup_cls(basename)
    sample = {
        "__key__": basename,
        "input.png": image,
        "target.cls": cls
    }
    sink.write(sample)
sink.close()
```

Since no encoder is used, if you want to be able to read this data with the default decoder, `image` must contain a byte string corresponding to a PNG image (as indicated by the ".png" extension on its dictionary key), and `cls` must contain an integer encoded in ASCII (as indicated by the ".cls" extension on its dictionary key).

# Writing Filters and Offline Augmentation

Webdataset can be used for filters and offline augmentation of datasets. Here is a complete example that pre-augments a shard and extracts class labels.


```python
def extract_class(data):
    # mock implementation
    return 0

def augment_wds(input, output, maxcount=999999999):
    src = (
        wds.Dataset(input)
        .decode("pil")
        .to_tuple("__key__", "jpg;png", "json")
        .map_tuple(identity, preproc, identity)
    )
    with wds.TarWriter(output) as dst:
        for key, image, data in islice(src, 0, maxcount):
            print(key)
            image = image.numpy().transpose(1, 2, 0)
            image -= amin(image)
            image /= amax(image)
            sample = {
                "__key__": key,
                "png": image,
                "cls": extract_class(data)
            }
            dst.write(sample)
```

Now run the augmentation pipeline:


```python
url = "http://storage.googleapis.com/nvdata-openimages/openimages-train-000000.tar"
url = f"pipe:curl -L -s {url} || true"
augment_wds(url, "_temp.tar", maxcount=5)
```


```python
e39871fd9fd74f55
f18b91585c4d3f3e
ede6e66b2fb59aab
ed600d57fcee4f94
ff47e649b23f446d

```

To verify that things worked correctly, let's look at the output file:


```bash
%%bash
%%bash
tar tf _temp.tar
```


```python
e39871fd9fd74f55.cls
e39871fd9fd74f55.png
f18b91585c4d3f3e.cls
f18b91585c4d3f3e.png
ede6e66b2fb59aab.cls
ede6e66b2fb59aab.png
ed600d57fcee4f94.cls
ed600d57fcee4f94.png
ff47e649b23f446d.cls
ff47e649b23f446d.png

```

If you want to preprocess the entire OpenImages dataset with a process like this, you can use your favorite job queueing or worflow system.

For example, using Dask, you could process all 554 shards in parallel using code like this:


```python
shards = braceexpand.braceexpand("{000000..000554}")
inputs = [f"gs://bucket/openimages-{shard}.tar" for shard in shards]
outputs = [f"gs://bucket2/openimages-augmented-{shard}.tar" for shard in shards]
results = [dask.delayed(augment_wds)(args) for args in zip(inputs, outputs)]
dask.compute(*results)
```

Note that the data is streaming from and to Google Cloud Storage buckets, so very little local storage is required on each worker.

For very large scale processing, it's easiest to submit separate jobs to a Kubernetes cluster using the Kubernetes `Job` template, or using a workflow engine like Argo.

# WebDataset = Composition of Iterable Datasets

The `webdataset` library is really just a collection of useful pipeline stages that you can put together in many different ways, and the `webdataset.Dataset` class is just a convenient wrapper for building up a pipeline.

A `Dataset` defined like:
    
```Python
wds.Dataset(url).shuffle(100).decode("pil")
```

is actually a composition of multiple classes:

```Python
dataset = DecodeProcessor(ShuffleProcessor(Dataset(url), 100))
```

or, equivalently:

```Python
raw = Dataset(url)
shuffled = ShuffleProcessor(raw, 100)
dataset = DecodeProcessor(shuffled)
```

Here, `Dataset`, `ShuffleProcessor`, and `DecodeProcessor` are just `IterableDataset` implementations that take other `IterableDataset` implementations as a source.

You can implement these processing classes directly by subclassing `IterableDataset`, but that's kind of repetitive too. In fact, at the heart of almost every `IterableDataset` is a function that maps an input iterator to an output iterator. Here is a simple processor that augments samples by adding noise:


```python
def augment_sample(source, noise=0.01):
    for inputs, targets in source:
        inputs += torch.randn_like(inputs) * noise
        yield inputs, targets
```

You can turn this into an `IterableDataset` by writing a wrapper:


```python
class AugmentSampleProcessor(IterableDataset):
    
    def __init__(self, source_dataset, noise=0.01):
        self.source_dataset = source_dataset
        self.noise = noise
        
    def __iter__(self):
        return augment_sample(iter(source_dataset), noise=self.noise)
    
    def __len__(self):
        return len(self.source_dataset)
```

Since this gets kind of tedious to write over and over again, the `webdataset` library just provides a wrapper that automates this for you:


```python
AugmentSampleProcessor = wds.Processor(augment_sample, noise=0.01)
isinstance(AugmentSampleProcessor, IterableDataset)
```




    True



And as a further shorthand, you can actually create processing chains with the `.then` method.


```python
dataset = (
    wds.WebDataset(url)
    .then(wds.shuffle, 100)
    .then(wds.decode, "pil")
    .then(augment_sample, noise=0.01)
)
```

Or you can compose a dataset using the `.compose` method:


```python
dataset = (
    wds.WebDataset(url)
    .then(wds.shuffle, 100)
    .then(wds.decode, "pil")
    .compose(AugmentSampleProcessor)
)
```

In fact, `webdataset.WebDataset` is just a shorthand for a simple processing pipeline:


```python
dataset = (
    wds.ShardList(url)
    .then(wds.url_opener)
    .then(wds.tar_file_expander)
    .then(wds.group_by_keys)
)
```


```python
next(iter(dataset)).keys()
```




    dict_keys(['__key__', 'jpg', 'json'])



The stages in this processing pipeline are:

- `ShardList` generates an iterator of URLs; you may need to modify this to generate different shard subsets for multinode computation
- `url_opener` opens each URL in turn and returns a bytestream
- `tarfile_expander` takes each bytestream in its input and generates an iterator over the tar files in that stream
- `group_by_keys` groups files together into training samples

In addition to those tar-file related processing functions, there are additional processing functions for common dataset operations:

- `info` -- provide info about training samples from an IterableDataset
- `shuffle` -- shuffle the IterableDataset
- `select` -- select samples from the IterableDataset
- `decode` -- apply decoding functions to each key-value pair in a sample
- `map` -- apply a mapping function to each sample
- `rename` -- rename the keys in key-value pairs in a sample
- `map_dict` -- apply functions to selected key-value pairs
- `to_tuple` -- convert dictionary-based samples to tuple-based samples
- `map_tuple` -- apply functions to the elements of each tuple-based sample
- `batched` -- batch samples together using a collation functions
- `unbatched` -- unbatch batched samples

Use `help(wds.batched)` etc. to get more information on each function.

Whether you prefer `WebDataset` or `Dataset` is a matter of style.

# Splitting Shards across Nodes and Workers

Unlike traditional PyTorch `Dataset` instances, `WebDataset` splits data across nodes at the shard level. This functionality is handled inside the `ShardList` class. To customize splitting data across nodes, you can either write your own `ShardList` function, or you can give the `ShardList` class a `splitter=` argument when you create it. The `splitter` function should select a subset of shards from the given list of shards based on the current node and worker ID.

# Data Sources

The `ShardList` class takes either a string or a list of URLs as an argument. If it is given a string, the string is expanded using the `braceexpand` library. So, the following are equivalent:

```Python
ShardList("dataset-{000..001}.tar")
ShardList(["dataset-000.tar", "dataset-001.tar"])
```

The url strings in a shard list are handled by default by the `webdataset.url_opener` filter. It recognizes three simple kinds of strings: "-", "/path/to/file", and "pipe:command":

- the string "-", referring to stdin
- a UNIX path, opened as a regular file
- a URL-like string with the schema "pipe:"; such URLs are opened with `subprocess.Popen`. For example:
    - `pipe:curl -s -L http://server/file` accesses a file via HTTP
    - `pipe:gsutil cat gs://bucket/file` accesses a file on GCS
    - `pipe:az cp --container bucket --name file --file /dev/stdout` accesses a file on Azure
    - `pipe:ssh host cat file` accesses a file via `ssh`

It might seem at first glance to be "more efficient" to use built-in Python libraries for accessing object stores rather than subprocesses, but efficient object store access from Python really requires spawning a separate process anyway, so this approach to accessing object stores is not only convenient, it also is as efficient as we can make it in Python.

# Related Libraries and Software

The [AIStore](http://github.com/NVIDIA/aistore) server provides an efficient backend for WebDataset; it functions like a combination of web server, content distribution network, P2P network, and distributed file system. Together, AIStore and WebDataset can serve input data from rotational drives distributed across many servers at the speed of local SSDs to many GPUs, at a fraction of the cost. We can easily achieve hundreds of MBytes/s of I/O per GPU even in large, distributed training jobs.

The [tarproc](http://github.com/tmbdev/tarproc) utilities provide command line manipulation and processing of webdatasets and other tar files, including splitting, concatenation, and `xargs`-like functionality.

The [tensorcom](http://github.com/tmbdev/tensorcom/) library provides fast three-tiered I/O; it can be inserted between [AIStore](http://github.com/NVIDIA/aistore) and [WebDataset](http://github.com/tmbdev/webdataset) to permit distributed data augmentation and I/O. It is particularly useful when data augmentation requires more CPU than the GPU server has available.

You can find the full PyTorch ImageNet sample code converted to WebDataset at [tmbdev/pytorch-imagenet-wds](http://github.com/tmbdev/pytorch-imagenet-wds)

# TODO

- refactor `autodecode.py`; allow for cascade of decoders
- add single file decompression to `autodecode.py`
- integrate Tensorcom library
- key rewriting / custom key grouping
