# Storage Manager

## Introduction

The Storage Manager is the glue between client and server sides. Every
object saved in the cache is allocated a *StoreEntry* structure. While
the object is being accessed, it also has a *MemObject* structure.

Squid can quickly locate cached objects because it keeps (in memory) a
hash table of all *StoreEntry*. The keys for the hash table are MD5
checksums of the objects URI. In addition there is also a storage policy
such as LRU that keeps track of the objects and determines the removal
order when space needs to be reclaimed. For the LRU policy this is
implemented as a doubly linked list.

For each object the *StoreEntry* maps to a cache\_dir and location via
sdirn and sfilen. For the "ufs" store this file number (sfilen) is
converted to a disk pathname by a simple modulo of L2 and L1, but other
storage drivers may map sfilen in other ways. A cache swap file consists
of two parts: the cache metadata, and the object data. Note the object
data includes the full HTTP reply---headers and body. The HTTP reply
headers are not the same as the cache metadata.

Client-side requests register themselves with a *StoreEntry* to be
notified when new data arrives. Multiple clients may receive data via a
single *StoreEntry*. For POST and PUT request, this process works in
reverse. Server-side functions are notified when additional data is read
from the client.

## Object storage

To be written...

## Object retrieval

To be written...

# Storage Interface

## Introduction

Traditionally, Squid has always used the Unix filesystem (UFS) to store
cache objects on disk. Over the years, the poor performance of UFS has
become very obvious. In most cases, UFS limits Squid to about 30-50
requests per second. Our work indicates that the poor performance is
mostly due to the synchronous nature of `open()` and `unlink()` system
calls, and perhaps thrashing of inode/buffer caches.

We want to try out our own, customized filesystems with Squid. In order
to do that, we need a well-defined interface for the bits of Squid that
access the permanent storage devices. We also require tighter control of
the replacement policy by each storage module, rather than a single
global replacement policy.

## Build structure

The storage types live in squid/src/fs/ . Each subdirectory corresponds
to the name of the storage type. When a new storage type is implemented
configure.in must be updated to autogenerate a Makefile in
squid/src/fs/$type/ from a Makefile.in file.

configure will take a list of storage types through the
*--enable-store-io* parameter. This parameter takes a list of space
seperated storage types. For example, --enable-store-io="ufs coss" .

Each storage type must create an archive file `in squid/src/fs/$type.a`
. This file is automatically linked into squid at compile time.

Each storefs must export a function named `storeFsSetup_$type()`. This
function is called at runtime to initialise each storage type. The list
of storage types is passed through `store_modules.sh` to generate the
initialisation function `storeFsSetup()`. This function lives in
`store_modules.c`.

An example of the automatically generated file:

``` cpp
/* automatically generated by ./store_modules.sh ufs coss
 * do not edit
 */
#include "squid.h"

extern STSETUP storeFsSetup_ufs;
extern STSETUP storeFsSetup_coss;
void storeFsSetup(void)
{
                storeFsAdd("ufs", storeFsSetup_ufs);
                storeFsAdd("coss", storeFsSetup_coss);
}
```

## Initialization of a storage type

Each storage type initializes through the `storeFsSetup_$type()`
function. The `storeFsSetup_$type()` function takes a single argument -
a `storefs_entry_t` pointer. This pointer references the storefs\_entry
to initialise. A typical setup function is as follows:

``` cpp
void
storeFsSetup_ufs(storefs_entry_t *storefs)
{
        assert(!ufs_initialised);
        storefs->parsefunc = storeUfsDirParse;
        storefs->reconfigurefunc = storeUfsDirReconfigure;
        storefs->donefunc = storeUfsDirDone;
        ufs_state_pool = memPoolCreate("UFS IO State data", sizeof(ufsstate_t));
        ufs_initialised = 1;
}
```

There are five function pointers in the storefs\_entry which require
initializing. In this example, some protection is made against the setup
function being called twice, and a memory pool is initialised for use
inside the storage module.

Each function will be covered below.

### done

``` cpp
typedef void
STFSSHUTDOWN(void);
```

This function is called whenever the storage system is to be shut down.
It should take care of deallocating any resources currently allocated.

``` cpp
typedef void
STFSPARSE(SwapDir *SD, int index, char *path);

typedef void
STFSRECONFIGURE(SwapDir *SD, int index, char *path);
```

These functions handle configuring and reconfiguring a storage
directory. Additional arguments from the cache\_dir configuration line
can be retrieved through calls to strtok() and GetInteger().

*STFSPARSE* has the task of initialising a new swapdir. It should parse
the remaining arguments on the cache\_dir line, initialise the relevant
function pointers and data structures, and choose the replacement
policy. *STFSRECONFIGURE* deals with reconfiguring an active swapdir. It
should parse the remaining arguments on the cache\_dir line and change
any active configuration parameters. The actual storage initialisation
is done through the *STINIT* function pointer in the SwapDir.

``` cpp
struct _SwapDir {
        char *type;                             /* Pointer to the store dir type string */
        int cur_size;                           /* Current swapsize in kb */
        int low_size;                           /* ?? */
        int max_size;                           /* Maximum swapsize in kb */
        char *path;                             /* Path to store */
        int index;                              /* This entry's index into the swapDir array */
        int suggest;                            /* Suggestion for UFS style stores (??) */
        size_t max_objsize;                     /* Maximum object size for this store */
        union {                                 /* Replacement policy-specific fields */
        #ifdef HEAP_REPLACEMENT
                struct {
                        heap *heap;
                } heap;
        #endif
                struct {
                        dlink_list list;
                        dlink_node *walker;
                } lru;
        } repl;
        int removals;
        int scanned;
        struct {
                unsigned int selected:1;        /* Currently selected for write */
                unsigned int read_only:1;       /* This store is read only */
        } flags;
        STINIT *init;                           /* Initialise the fs */
        STNEWFS *newfs;                         /* Create a new fs */
        STDUMP *dump;                           /* Dump fs config snippet */
        STFREE *freefs;                         /* Free the fs data */
        STDBLCHECK *dblcheck;                   /* Double check the obj integrity */
        STSTATFS *statfs;                       /* Dump fs statistics */
        STMAINTAINFS *maintainfs;               /* Replacement maintainence */
        STCHECKOBJ *checkob;                    /* Check if the fs will store an object, and get the FS load */
        /* These two are notifications */
        STREFOBJ *refobj;                       /* Reference this object */
        STUNREFOBJ *unrefobj;                   /* Unreference this object */
        STCALLBACK *callback;                   /* Handle pending callbacks */
        STSYNC *sync;                           /* Sync the directory */
        struct {
                STOBJCREATE *create;            /* Create a new object */
                STOBJOPEN *open;                /* Open an existing object */
                STOBJCLOSE *close;              /* Close an open object */
                STOBJREAD *read;                /* Read from an open object */
                STOBJWRITE *write;              /* Write to a created object */
                STOBJUNLINK *unlink;            /* Remove the given object */
        } obj;
        struct {
                STLOGOPEN *open;                /* Open the log */
                STLOGCLOSE *close;              /* Close the log */
                STLOGWRITE *write;              /* Write to the log */
                struct {
                        STLOGCLEANOPEN *open;   /* Open a clean log */
                        STLOGCLEANWRITE *write; /* Write to the log */
                        void *state;            /* Current state */
                } clean;
        } log;
        void *fsdata;                           /* FS-specific data */
};
```

## Operation of a storage module

Squid understands the concept of multiple diverse storage directories.
Each storage directory provides a caching object store, with object
storage, retrieval, indexing and replacement.

Each open object has associated with it a *storeIOState* object. The
*storeIOState* object is used to record the state of the current object.
Each *storeIOState* can have a storage module specific data structure
containing information private to the storage module.

``` cpp
struct _storeIOState {
        sdirno swap_dirn;               /* SwapDir index */
        sfileno swap_filen;             /* Unique file index number */
        StoreEntry *e;                  /* Pointer to parent StoreEntry */
        mode_t mode;                    /* Mode - O_RDONLY or O_WRONLY */
        size_t st_size;                 /* Size of the object if known */
        off_t offset;                   /* current _on-disk_ offset pointer */
        STFNCB *file_callback;          /* called on delayed sfileno assignments */
        STIOCB *callback;               /* IO Error handler callback */
        void *callback_data;            /* IO Error handler callback data */
        struct {
                STRCB *callback;        /* Read completion callback */
                void *callback_data;    /* Read complation callback data */
        } read;
        struct {
                unsigned int closing:1; /* debugging aid */
        } flags;
        void *fsstate;                  /* pointer to private fs state */
};
```

Each *SwapDir* has the concept of a maximum object size. This is used as
a basic hint to the storage layer in first choosing a suitable
*SwapDir*. The checkobj function is then called for suitable candidate
*SwapDirs* to find out whether it wants to store a given *StoreEntry*. A
*maxobjsize* of -1 means 'any size'.

The specific filesystem operations listed in the SwapDir object are
covered below.

### initfs

``` cpp
typedef void STINIT(SwapDir *SD);
```

Initialise the given *SwapDir*. Operations such as verifying and
rebuilding the storage and creating any needed bitmaps are done here.

### newfs

``` cpp
typedef void
STNEWFS(SwapDir *SD);
```

Called for each configured *SwapDir* to perform filesystem
initialisation. This happens when '-z' is given to squid on the command
line.

### dumpfs

``` cpp
typedef void
STDUMP(StoreEntry *e, SwapDir *SD);
```

Dump the FS specific configuration data of the current *SwapDir* to the
given *StoreEntry*. Used to grab a configuration file dump from th
*cachemgr* interface.

Note: The printed options should start with a space character to
separate them from the cache\_dir path.

### freefs

``` cpp
typedef void
STFREE(SwapDir *SD);
```

Free the *SwapDir* filesystem information. This routine should
deallocate *SD-\>fsdata*.

### doublecheckfs

``` cpp
typedef int
STDBLCHECK(SwapDir *SD, StoreEntry *e);
```

Double-check the given object for validity. Called during rebuild if the
'-S' flag is given to squid on the command line. Returns 1 if the object
is indeed valid, and 0 if the object is found invalid.

### statfs

``` cpp
typedef void
STSTATFS(SwapDir *SD, StoreEntry *e);
```

Called to retrieve filesystem statistics, such as usage, load and
errors. The information should be appended to the passed *StoreEntry* e.

### maintainfs

``` cpp
typedef void
STMAINTAINFS(SwapDir *SD);
```

Called periodically to replace objects. The active replacement policy
should be used to timeout unused objects in order to make room for new
objects.

### callback

``` cpp
typedef void
STCALLBACK(SwapDir *SD);
```

This function is called inside the comm\_select/comm\_poll loop to
handle any callbacks pending.

### sync

``` cpp
typedef void
STSYNC(SwapDir *SD);
```

This function is called whenever a sync to disk is required. This
function should not return until all pending data has been flushed to
disk.

### parse/reconfigure

### checkobj

``` cpp
typedef int
STCHECKOBJ(SwapDir *SD, const StoreEntry *e);
```

Called by `storeDirSelectSwapDir()` to determine whether the *SwapDir*
will store the given *StoreEntry* object. If the *SwapDir* is not
willing to store the given *StoreEntry* -1 should be returned.
Otherwise, a value between 0 and 1000 should be returned indicating the
current IO load. A value of 1000 indicates the *SwapDir* has an IO load
of 100%. This is used by `storeDirSelectSwapDir()` to choose the
*SwapDir* with the lowest IO load.

### referenceobj

``` cpp
typedef void
STREFOBJ(SwapDir *SD, StoreEntry *e);
```

Called whenever an object is locked by `storeLockObject()`. It is
typically used to update the objects position in the replacement policy.

### unreferenceobj

``` cpp
typedef void
STUNREFOBJ(SwapDir *SD, StoreEntry *e);
```

Called whenever the object is unlocked by `storeUnlockObject()` and the
lock count reaches 0. It is also typically used to update the objects
position in the replacement policy.

### createobj

``` cpp
typedef storeIOState *
STOBJCREATE(SwapDir *SD, StoreEntry *e, STFNCB *file_callback, STIOCB *io_callback, void *io_callback_data);
```

Create an object in the *SwapDir* \*SD. *file\_callback* is called
whenever the filesystem allocates or reallocates the *swap\_filen*. Note
- *STFNCB* is called with a generic cbdata pointer, which points to the
*StoreEntry* e. The *StoreEntry* should not be modified EXCEPT for the
replacement policy fields.

The IO callback should be called when an error occurs and when the
object is closed. Once the IO callback is called, the *storeIOState*
becomes invalid.

*STOBJCREATE* returns a *storeIOState* suitable for writing on sucess,
or NULL if an error occurs.

### openobj

``` cpp
typedef storeIOState *
STOBJOPEN(SwapDir *SD, StoreEntry *e, STFNCB *file_callback, STIOCB *io_callback, void *io_callback_data);
```

Open the *StoreEntry* in *SwapDir* \*SD for reading. Much the same is
applicable from *STOBJCREATE*, the major difference being that the data
passed to *file\_callback* is the relevant *store\_client* .

### closeobj

``` cpp
typedef void
STOBJCLOSE(SwapDir *SD, storeIOState *sio);
```

Close an opened object. The *STIOCB* callback should be called at the
end of this routine.

### readobj

``` cpp
typedef void
STOBJREAD(SwapDir *SD, storeIOState *sio, char *buf, size_t size, off_t offset, STRCB *read_callback, void *read_callback_data);
```

Read part of the object of into *buf*. It is safe to request a read when
there are other pending reads or writes. *STRCB* is called at
completion.

If a read operation fails, the filesystem layer notifies the calling
module by calling the *STIOCB* callback with an error status code.

### writeobj

``` cpp
typedef void
STOBJWRITE(SwapDir *SD, storeIOState *sio, char *buf, size_t size, off_t offset, FREE *freefunc);
```

Write the given block of data to the given store object. *buf* is
allocated by the caller. When the write is complete, the data is freed
through *free\_func*.

If a write operation fails, the filesystem layer notifies the calling
module by calling the *STIOCB* callback with an error status code.

### unlinkobj

``` cpp
typedef void
STOBJUNLINK(SwapDir *, StoreEntry *);
```

Remove the *StoreEntry* e from the *SwapDir* SD and the replacement
policy.

## Store IO calls

These routines are used inside the storage manager to create and
retrieve objects from a storage directory.

### storeCreate()

``` cpp
storeIOState *
storeCreate(StoreEntry *e, STIOCB *file_callback, STIOCB *close_callback, void * callback_data)
```

`storeCreate` is called to store the given *StoreEntry* in a storage
directory.

`callback` is a function that will be called either when an error is
encountered, or when the object is closed (by calling `storeClose()`).
If the open request is successful, there is no callback. The calling
module must assume the open request will succeed, and may begin reading
or writing immediately.

`storeCreate()` may return NULL if the requested object can not be
created. In this case the `callback` function will not be called.

### storeOpen()

``` cpp
storeIOState *
storeOpen(StoreEntry *e, STFNCB * file_callback, STIOCB * callback, void *callback_data)
```

`storeOpen` is called to open the given *StoreEntry* from the storage
directory it resides on.

`callback` is a function that will be called either when an error is
encountered, or when the object is closed (by calling `storeClose()`).
If the open request is successful, there is no callback. The calling
module must assume the open request will succeed, and may begin reading
or writing immediately.

`storeOpen()` may return NULL if the requested object can not be
openeed. In this case the `callback` function will not be called.

### storeRead()

``` cpp
void
storeRead(storeIOState *sio, char *buf, size_t size, off_t offset, STRCB *callback, void *callback_data)
```

`storeRead()` is more complicated than the other functions because it
requires its own callback function to notify the caller when the
requested data has actually been read. *buf* must be a valid memory
buffer of at least *size* bytes. *offset* specifies the byte offset
where the read should begin. Note that with the Swap Meta Headers
prepended to each cache object, this offset does not equal the offset
into the actual object data.

The caller is responsible for allocating and freeing *buf* .

### storeWrite()

``` cpp
void
storeWrite(storeIOState *sio, char *buf, size_t size, off_t offset, FREE *free_func)
```

`storeWrite()` submits a request to write a block of data to the disk
store. The caller is responsible for allocating *buf*, but since there
is no per-write callback, this memory must be freed by the lower
filesystem implementation. Therefore, the caller must specify the
*free\_func* to be used to deallocate the memory.

If a write operation fails, the filesystem layer notifies the calling
module by calling the *STIOCB* callback with an error status code.

### storeUnlink()

``` cpp
void
storeUnlink(StoreEntry *e)
```

`storeUnlink()` removes the cached object from the disk store. There is
no callback function, and the object does not need to be opened first.
The filesystem layer will remove the object if it exists on the disk.

### storeOffset()

``` cpp
off_t
storeOffset(storeIOState *sio)
```

`storeOffset()` returns the current \_ondisk\_ offset. This is used to
determine how much of an objects memory can be freed to make way for
other in-transit and cached objects. You must make sure that the
*storeIOState-\>offset* refers to the ondisk offset, or undefined
results will occur. For reads, this returns the current offset of
successfully read data, not including queued reads.

## Callbacks

### STIOCB callback

``` cpp
void
stiocb(void *data, int errorflag, storeIOState *sio)
```

The *stiocb* function is passed as a parameter to `storeOpen()`. The
filesystem layer calls *stiocb* either when an I/O error occurs, or when
the disk object is closed.

*errorflag* is one of the following:

``` cpp
#define DISK_OK                   (0)
#define DISK_ERROR               (-1)
#define DISK_EOF                 (-2)
#define DISK_NO_SPACE_LEFT       (-6)
```

Once the The *stiocb* function has been called, the *sio* structure
should not be accessed further.

### STRCB callback

``` cpp
void
strcb(void *data, const char *buf, size_t len)
```

The *strcb* function is passed as a parameter to `storeRead()`. The
filesystem layer calls *strcb* after a block of data has been read from
the disk and placed into *buf*. *len* indicates how many bytes were
placed into *buf*. The *strcb* function is only called if the read
operation is successful. If it fails, then the *STIOCB* callback will be
called instead.

## State Logging

These functions deal with state logging and related tasks for a squid
storage system. These functions are used (called) in `store_dir.c`.

Each storage system must provide the functions described in this
section, although it may be a no-op (null) function that does nothing.
Each function is accessed through a function pointer stored in the
*SwapDir* structure:

``` cpp
    struct _SwapDir {
        ...
        STINIT *init;
        STNEWFS *newfs;
        struct {
            STLOGOPEN *open;
            STLOGCLOSE *close;
            STLOGWRITE *write;
            struct {
                STLOGCLEANOPEN *open;
                STLOGCLEANWRITE *write;
                void *state;
            } clean;
        } log;
        ....
    };
```

### log.open()

``` cpp
void
STLOGOPEN(SwapDir *);
```

The `log.open` function, of type *STLOGOPEN*, is used to open or
initialize the state-holding log files (if any) for the storage system.
For UFS this opens the *swap.state* files.

The `log.open` function may be called any number of times during Squid's
execution. For example, the process of rotating, or writing clean
logfiles closes the state log and then re-opens them. A *squid -k
reconfigure* does the same.

### log.close()

``` cpp
void
STLOGCLOSE(SwapDir *);
```

The `log.close` function, of type *STLOGCLOSE*, is obviously the
counterpart to `log.open`. It must close the open state-holding log
files (if any) for the storage system.

### log.write()

``` cpp
void
STLOGWRITE(const SwapDir *, const StoreEntry *, int op);
```

The `log.write` function, of type *STLOGWRITE*, is used to write an
entry to the state-holding log file. The *op* argument is either
*SWAP\_LOG\_ADD* or *SWAP\_LOG\_DEL*. This feature may not be required
by some storage systems and can be implemented as a null-function
(no-op).

### log.clean.start()

``` cpp
int
STLOGCLEANSTART(SwapDir *);
```

The `log.clean.start` function, of type *STLOGCLEANSTART*, is used for
the process of writing "clean" state-holding log files. The
clean-writing procedure is initiated by the *squid -k rotate* command.
This is a special case because we want to optimize the process as much
as possible. This might be a no-op for some storage systems that don't
have the same logging issues as UFS.

The *log.clean.state* pointer may be used to keep state information for
the clean-writing process, but should not be accessed by upper layers.

### log.clean.nextentry()

``` cpp
StoreEntry *
STLOGCLEANNEXTENTRY(SwapDir *);
```

Gets the next entry that is a candidate for the clean log.

Returns NULL when there is no more objects to log

### log.clean.write()

``` cpp
void
STLOGCLEANWRITE(SwapDir *, const StoreEntry *);
```

The `log.clean.write` function, of type *STLOGCLEANWRITE*, writes an
entry to the clean log file (if any).

### log.clean.done()

``` cpp
void
STLOGCLEANDONE(SwapDir *);
```

Indicates the end of the clean-writing process and signals the storage
system to close the clean log, and rename or move them to become the
official state-holding log ready to be opened.

## Replacement policy implementation

The replacement policy can be updated during
STOBJREAD/STOBJWRITE/STOBJOPEN/ STOBJCLOSE as well as STREFOBJ and
STUNREFOBJ. Care should be taken to only modify the relevant replacement
policy entries in the StoreEntry. The responsibility of replacement
policy maintainence has been moved into each SwapDir so that the storage
code can have tight control of the replacement policy. Cyclic
filesystems such as COSS require this tight coupling between the storage
layer and the replacement policy.

## Removal policy API

The removal policy is responsible for determining in which order objects
are deleted when Squid needs to reclaim space for new objects. Such a
policy is used by a object storage for maintaining the stored objects
and determining what to remove to reclaim space for new objects.
(together they implements a replacement policy)

### API

It is implemented as a modular API where a storage directory or memory
creates a policy of choice for maintaining it's objects, and modules
registering to be used by this API.

#### createRemovalPolicy()

``` cpp
RemovalPolicy policy = createRemovalPolicy(cons char *type, cons char *args)
```

Creates a removal policy instance where object priority can be
maintained

The returned RemovalPolicy instance is cbdata registered

#### policy.Free()

``` cpp
policy->Free(RemovalPolicy *policy)
```

Destroys the policy instance and frees all related memory.

#### policy.Add()

``` cpp
policy->Add(RemovalPolicy *policy, StoreEntry *, RemovalPolicyNode *node)
```

Adds a StoreEntry to the policy instance.

datap is a pointer to where policy specific data can be stored for the
store entry, currently the size of one (void \*) pointer.

#### policy.Remove()

``` cpp
policy->Remove(RemovalPolicy *policy, StoreEntry *, RemovalPolicyNode *node)
```

Removes a StoreEntry from the policy instance out of policy order. For
example when an object is replaced by a newer one or is manually purged
from the store.

datap is a pointer to where policy specific data is stored for the store
entry, currently the size of one (void \*) pointer.

#### policy.Referenced()

``` cpp
policy->Referenced(RemovalPolicy *policy, const StoreEntry *, RemovalPolicyNode *node)
```

Tells the policy that a StoreEntry is going to be referenced. Called
whenever a entry gets locked.

node is a pointer to where policy specific data is stored for the store
entry, currently the size of one (void \*) pointer.

#### policy.Dereferenced()

``` cpp
policy->Dereferenced(RemovalPolicy *policy, const StoreEntry *, RemovalPolicyNode *node)
```

Tells the policy that a StoreEntry has been referenced. Called when an
access to the entry has finished.

node is a pointer to where policy specific data is stored for the store
entry, currently the size of one (void \*) pointer.

#### policy.WalkInit()

``` cpp
RemovalPolicyWalker walker = policy->WalkInit(RemovalPolicy *policy)
```

Initiates a walk of all objects in the policy instance. The objects is
returned in an order suitable for using as reinsertion order when
rebuilding the policy.

The returned RemovalPolicyWalker instance is cbdata registered

Note: The walk must be performed as an atomic operation with no other
policy actions intervening, or the outcome will be undefined.

#### walker.Next()

``` cpp
const StoreEntry *entry = walker->Next(RemovalPolicyWalker *walker)
```

Gets the next object in the walk chain

Return NULL when there is no further objects

#### walker.Done()

``` cpp
walker->Done(RemovalPolicyWalker *walker)
```

Finishes a walk of the maintained objects, destroys walker.

#### policy.PurgeInit()

``` cpp
RemovalPurgeWalker purgewalker = policy->PurgeInit(RemovalPolicy *policy, int max_scan)
```

Initiates a search for removal candidates. Search depth is indicated by
max\_scan.

The returned RemovalPurgeWalker instance is cbdata registered

Note: The walk must be performed as an atomic operation with no other
policy actions intervening, or the outcome will be undefined.

#### purgewalker.Next()

``` cpp
StoreEntry *entry = purgewalker->Next(RemovalPurgeWalker *purgewalker)
```

Gets the next object to purge. The purgewalker will remove each returned
object from the policy.

It is the polices responsibility to verify that the object isn't locked
or otherwise prevented from being removed. What this means is that the
policy must not return objects where storeEntryLocked() is true.

Return NULL when there is no further purgeable objects in the policy.

#### purgewalker.Done()

``` cpp
purgewalker->Done(RemovalPurgeWalker *purgewalker)
```

Finishes a walk of the maintained objects, destroys walker and restores
the policy to it's normal state.

#### policy.Stats()

``` cpp
purgewalker->Stats(RemovalPurgeWalker *purgewalker, StoreEntry *entry)
```

Appends statistics about the policy to the given entry.

### Source layout

Policy implementations resides in src/repl/\<name\>/, and a make in such
a directory must result in a object archive src/repl/\<name\>.a
containing all the objects implementing the policy.

### Internal structures

#### RemovalPolicy

``` cpp
typedef struct _RemovalPolicy RemovalPolicy;
struct _RemovalPolicy {
    char *_type;
    void *_data;
    void (*add)(RemovalPolicy *policy, StoreEntry *);
    ... /* see the API definition above */
};
```

The \_type member is mainly for debugging and diagnostics purposes, and
should be a pointer to the name of the policy (same name as used for
creation)

The \_data member is for storing policy specific information.

#### RemovalPolicyWalker

``` cpp
typedef struct _RemovalPolicyWalker RemovalPolicyWalker;
struct _RemovalPolicyWalker {
    RemovalPolicy *_policy;
    void *_data;
    StoreEntry *(*next)(RemovalPolicyWalker *);
    ... /* see the API definition above */
};
```

#### RemovalPolicyNode

``` cpp
typedef struct _RemovalPolicyNode RemovalPolicyNode;
struct _RemovalPolicyNode {
    void *data;
};
```

Stores policy specific information about a entry. Currently there is
only space for a single pointer, but plans are to maybe later provide
more space here to allow simple policies to store all their data
"inline" to preserve some memory.

### Policy registration

Policies are automatically registered in the Squid binary from the
policy selection made by the user building Squid. In the future this
might get extended to support loadable modules. All registered policies
are available to object stores which wishes to use them.

### Policy instance creation

Each policy must implement a "create/new" function `RemovalPolicy *
createRemovalPolicy_<name>(char *arguments)` This function creates the
policy instance and populates it with at least the API methods
supported. Currently all API calls are mandatory, but the policy
implementation must make sure to NULL fill the structure prior to
populating it in order to assure future API compability.

It should also populate the \_data member with a pointer to policy
specific data.

### Walker

When a walker is created the policy populates it with at least the API
methods supported. Currently all API calls are mandatory, but the policy
implementation must make sure to NULL fill the structure prior to
populating it in order to assure future API compatibility.

### Design notes/bugs

The RemovalPolicyNode design is incomplete/insufficient. The intention
was to abstract the location of the index pointers from the policy
implementation to allow the policy to work on both on-disk and memory
caches, but unfortunately the purge method for HEAP based policies needs
to update this, and it is also preferable if the purge method in general
knows how to clear the information. I think the agreement was that the
current design of tightly coupling the two together on one StoreEntry is
not the best design possible.

It is debated if the design in having the policy index control the clean
index writes is the correct approach. Perhaps not. Perhaps a more
appropriate design is probably to do the store indexing completely
outside the policy implementation (i.e. using the hash index), and only
ask the policy to dump it's state somehow.

The Referenced/Dereferenced() calls is today mapped to lock/unlock which
is an approximation of when they are intended to be called. However, the
real intention is to have Referenced() called whenever an object is
referenced, and Dereferenced() only called when the object has actually
been used for anything good.

# Forwarding Selection

To be written...