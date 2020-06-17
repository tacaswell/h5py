Working theory of HDF5 internals
--------------------------------

In `h5py` we provide a very high-level abstraction of an hdf5 file: a
nested dictionaries of either more dictionaries or numpy arrays and
each element may have a dictionary of attributes attached to it.  With
that understanding (and a bit of spelling for the quirks of the API)
you can make reasonable use of `h5py` / hdf5.  However the abstraction
is a bit leaky and having a better theory of what HDF5 is doing under
the hood can make it easier to understand some of the behaviors ("why
is this access pattern slow"?, "why didn't the file get smaller after
I deleted a dataset", etc).  This is not meant to be a precise description
of HDF5 internals, but an effective or working theory to help reason about
the observed behavior.


Internally HDF5 is can be thought of as a file system (with groups
being the "folders", datasets being the "files", and links being links).


Internally and HDF5 file is made up of groups (which contain other
groups or datasets and may have attributes), datasets (which are best
thought of as ND arrays and may have attributes). and links (which are
pointers to the other two things).  This maps very closely to our
nested-dictionaries-of-numpyarrays model above.  HDF5 stores the
layout and the payload separately so there is a tree that specifies
what groups are nested in what other groups and what dataset is in
what group etc.  That structure stores offsets off to other places in
the file to where the data is actually encoded.  On the plus this
means that if you want to change the size of a dataset or add datasets
to a group you only have to update a small amount of data, however
this is the reason that deleting a dataset does not shrink the file on
disk.  If the data were stored inline, then changing the size of
something that appears early in the file would require shift
everything else in the file, which for large files can be absurdly
expensive ("I just deleted a 10kb data set that happened to be early
in the file, lets read and re-write 100GB!")


Turning to just the datasets, the original target use cases for hdf5
were cases where you had relatively low cardinality of relatively
large (larger than available memory) high-dimensional arrays that you
wanted to be able to do efficient regional access on.


Say we have a (1k by 1k by 1k x 1k) array that we want to be able to
pull a (100 by 100 by 10 by 1) hyper slab out of.  The key point here is I am pulling
out a contiguous volume in the data


￼￼
tacaswell  14 days ago
If you were to have enough ram to pull the whole dataset up doing a slice like that would be pretty slow to do because the data locality is quite bad (all memory is linear!)
￼￼
tacaswell  14 days ago
the way hdf5 handles this is with the notion of "chunking" that breaks your huge array up into an array of smaller arrays
￼￼
tacaswell  14 days ago
and the "dataset" turns into another set of offsets that knows "this volume of the data set is written at this offset in the file"
￼￼
tacaswell  14 days ago
these chunks end up being the "quantum" of data that the rest of hdf5 is built around
￼￼
tacaswell  14 days ago
if you apply compression to your datasets it is applied at the chunk level
￼￼
tacaswell  14 days ago
the chunk is the minimal amount of data that hdf5 can read from disk at a time and if you are doing compression (and maybe in all cases?) the minimum amount it can write to disk
￼￼
tacaswell  14 days ago
if you can arrange for you chunksize to match your access patterns exactly you can get some big performance wins (as you never have to read more than 1 chunk at a time).
￼￼
Pete Jemian  14 days ago
This is the detail in HDF5 that people misunderstand, leading to performance problems.
￼￼
1
￼
￼￼
Pete Jemian  14 days ago
How to chunk their data for performance.
￼￼
tacaswell  14 days ago
if you ask hdf5 for a volume that spans multiple chunks the library takes care of pulling up all of them, picking out the data you want and then assembling it for you into a nice contiguous array
￼￼
tacaswell  14 days ago
you can imagine some truely horrible access patterns here, say you have a (1M, 10) dataset chunked into (1k, 10) chunks
￼￼
tacaswell  14 days ago
if you then ask to read ds[::1_000] that is going to read the entire file from disk!
￼￼
tacaswell  14 days ago
it will be smart about not using all of your ram, but you will pay the disk I/O
￼￼
tacaswell  14 days ago
libhdf5 also takes care of type conversion for you.  It has a notion of "on disk type" and "in memory type".  When you read data out you say "please read out this selection from that data set into this array of type <memory type>"
￼￼
1
￼
￼￼
tacaswell  14 days ago
which transparently handles conversion to/from weird bit-depths, little endian vs big endian etc (edited)
￼￼
1
￼
￼￼
tacaswell  14 days ago
hdf5 also has the ability to have "external links" and "virtual datasets"
￼￼
Pete Jemian  14 days ago
external links are how Diamond handles fast frame rates from area detectors
￼￼
tacaswell  14 days ago
the first lets you say "this group or dataset in this file is actually stored as that group or dataset in some other hdf5 file"
￼￼
tacaswell  14 days ago
the second lets you say "the top left of this dataset is that data set in file A, the top right in that dataset in file B. ..."
￼￼
tacaswell  14 days ago
all of this requires a fair amount of careful book keeping to stay consistent
￼￼
tacaswell  14 days ago
say you are adding a new dataset, you need to do two things:
update the pointers to say where in the file the new data will be stored
actually write the binary stream of the (post conversion post compression) data to that location in the file
￼￼
tacaswell  14 days ago
if you only do 1, but not 2 and then come back to the file later it is going to fail
￼￼
tacaswell  14 days ago
if you are trying to write to the same file from multiple processes you are definitly going to mess up the bookkeeping
￼￼
tacaswell  14 days ago
if you try to put hdf5 files on a non-POSIX compliant filesystem (like an nfs mount) you are going to eventually have problems (due to not having the invariant that reading a file after a write just completed will return to you the content you just wrote instead of something stale
￼￼
tacaswell  14 days ago
the big change that hdf5group did for SWMR was to put some journaling / locking around steps 1 and 2.  If enabled the process is now "write 1, mark it is not 'in force', write 2, mark 1 as 'in force'" so it is safe to have many readers looking at a shared file that a single process can write to
￼￼
tacaswell  14 days ago
if you want to move away from hdf5 you have to have an answer for all of these features (chunking, transparent volume selection, compression, type conversion, links, ...) and at some point you are going to re-write libhdf5 but with out 20 years of engineering effort
￼￼
1
￼
￼￼
tacaswell  14 days ago
the most compelling citicism of hdf5 is the requirement that you have a posix file system which makes playing with object stores very hard
￼￼
Juliane Reinhardt  14 days ago
Thanks for the many details @tacaswell
￼￼
Dylan McReynolds  14 days ago
I just learned a lot.
￼￼
1
￼
￼￼
tacaswell  14 days ago
(on typing break for group meeting, will have more about how to handle object stores)
￼￼
2
￼
￼￼
Pete Jemian  14 days ago
Tom, I admire your depth of knowledge.
￼￼
tacaswell  14 days ago
for addressing this there are a couple of approaches.  One is h5srv / krita which is a server(s) that sit in front of your files and manages the "eventual consistency" of object store.  Those ideas got folded into the the VOL (virtual object layer) concept that is in hdf5 1.12 that (in very rough terms) uses pointers to blobs in a object store rather than pointers to offsets in a file
￼￼
tacaswell  14 days ago
zarr more-or-less does the same thing by using a directory tree of binary blobs (one per chunk) + a json file of meta-data
￼￼
tacaswell  14 days ago
in the case of posix and a blob per chunk if you are on an object store
￼￼
tacaswell  14 days ago
that gets you multi-reader, concurrent read/write, and multi-writer "for free" so long as two writers don't try to update the same chunk at the same time (to get multi-writer with hdf5 you have to use MPI, I have not done this so I'm not sure if hdf5 will manage any locking for you or if you just have to be careful to give each rank its own area of the dataset)
￼￼
tacaswell  14 days ago
tiledb does something slightly cooler, it writes time-stamped files of sparse arrays to a directory / object store
￼￼
tacaswell  14 days ago
and then when you access it has efficient ways of layering the full dataset and occasional consolidation steps of the layers into fewer files/objects
￼￼
tacaswell  14 days ago
if you have data that is much more table like you should be looking at things like parquet
￼￼
tacaswell  14 days ago
and if you really need efficient row-wise access of relatively small (1k) vectors, a database may be what you are looking for
￼￼
tacaswell  14 days ago
as a point of reference, I am one of the core maintainers of h5py
￼￼
tacaswell  14 days ago
</rant>
￼￼
Bjoern Enders  14 days ago
￼ ￼ chapeau!
