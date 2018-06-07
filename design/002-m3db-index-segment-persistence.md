# Proposal: Index Segment Flush and Bootstrap

Author(s): Rob Skillington, Prateek Rungta, Jerome Froelich
Last updated: Friday April 20, 2018
Discussion at [github.com/m3db/proposal/issues/2](https://github.com/m3db/proposal/issues/2).

## Abstract

This proposal outlines a change to bootstrap index segments by the database from segment files that get flushed to disk combined with series in the commit log that aren’t covered by any flushed segments.

## Background

M3DB uses [m3ninx](https://github.com/m3db/m3ninx) for reverse indexing and currently keeps segments of indexed IDs for a given time window or "block".  The database currently does not flush or bootstrap the segments that are created in memory, hence today restarting a database node will lose its index contents.

The proposed change outlines adding flush and bootstrap capabilities for metrics indexed by the database namespace index that currently reside just as segments in memory.

## Proposal

### File system layout

M3DB will keep an index "fileset volume" which will be a collection of files per time window “block” that can contain many segment file sets (each segment is a set of files itself), that when all segment filesets are read together will comprise the full set of indexed documents for a given time window “block”.  They will be laid out as such, the paths in bold delineate the new file directory paths:

```
<m3db-default-path>/

	commitlogs/

		// current commit logs

	data/

		<namespace>/

			<shard>/

				// current data block filesets

	snapshots/

		<namespace>/

			<shard>/

				// current data snapshot filesets


	index/

		data/

			<namespace>/

				// current index block filesets**

		snapshots/

			<namespace>/

				// current index snapshot filesets
```

### Flush and snapshot lifecycle

The flush lifecycle for persisting index segments to disk will be purely additive to the existing flush lifecycle for time series data in the database.  When the storage flush manager calls flush per namespace, it will the consequently call flush index per namespace, before it calls snapshot per namespace and then consequently snapshot index per namespace.  Both the flush index and snapshot index calls will be newly introduced calls that happen at the same cadence as the existing flush and snapshot calls.  

Before calling flush index on each namespace, the appropriate index block starts will be calculated using the namespace’s index retention options and for each potential block start the namespace’s index will be queried whether it needs a flush.

Before calling snapshot on the index for each namespace, the appropriate block start that could be snapshotted will be calculated using the namespace’s index retention options and will then call snapshot index consequently with that block start, if and only if the previous block start has already been flushed successfully.  This is how the logic for determining which block start to snapshot for time series data occurs today.

### Bootstrap lifecycle

The bootstrap lifecycle will be independent to the call chain that occurs for bootstrapping time series data.  Each bootstrapper will add a method to also read index metadata for a time range, alongside the current read data call for a time range.  This way all the splitting of time ranges, orchestration of executing the bootstrappers and merging of bootstrap result can stay in place.  The bootstrap process just needs to be run twice, once in read data mode and then again in read index mode.  

When bootstrapping occurs the bootstrap run options will be extended to specify whether bootstrapping for index data is enabled.  

Bootstrapper sources will be asked to return a new instance each time a bootstrapper run is executed, it will be used for a single bootstrap that returns both data and index metadata when indexing is enabled.  This will allow for the bootstrappers to reuse resources, such as the series metadata already read for series during a time window in the commit log bootstrapper, or similarly series metadata read from the peers bootstrapper, by keeping the data accessible on the bootstrapper source instance itself between the read data and read index metadata calls.

## Rationale

Since most of the code to implement the proposal will be largely the structure and lifecycle of the flush and bootstrap calls, we have opted to design and implement a forward looking filesystem schema that should support ongoing changes to the format of the segment files itself (which should not account for a large amount of code).  

This should allow relatively painless upgrades as new segment types are added.  However it is called out that it will not be 100% future proof for backwards/forwards compatibility guarantees and is a best estimate to keep the code structure largely the same to avoid too much structural change later.

## Implementation

### File system

All the existing types prefixed with Fileset or similar will be renamed to DataFileset, etc to distinguish between the new types.

Here are the new types corresponding interfaces:

```
type FileSetFileIdentifier struct {

	FileSetContentType persist.FileSetContentType

	Namespace          ident.ID

	BlockStart         time.Time

	// Only required for data content files

	Shard uint32

	// Only required for snapshot files

	Index int

}

type IndexWriterSnapshotOptions struct {

	SnapshotTime time.Time
}

type IndexWriterOpenOptions struct {

	Identifier  IndexFilesetFileIdentifier

	BlockSize   time.Duration

	FilesetType persist.FileSetType

	// Only used when writing snapshot files

	Snapshot IndexWriterSnapshotOptions

}

type IndexFileSetWriter interface {

	io.Closer

	Open(opts IndexWriterOpenOptions) error

	WriteSegmentFileSet(segmentFileSet IndexSegmentFileSet) error

}

type IndexSegmentFileSet interface {

	SegmentType() SegmentType

MajorVersion() int

MinorVersion() int

	SegmentMetadata() []byte

	Files() []IndexSegmentFile

}

type IndexSegmentFile interface {

	io.Reader

	io.Closer

	SegmentFileType() SegmentFileType

	

	// Bytes will be valid until the segment file is closed.

	Bytes() ([]byte, error)

}

type SegmentType string

type SegmentFileType string

type IndexReaderOpenOptions struct {

	Identifier  IndexFilesetFileIdentifier

	FilesetType persist.FileSetType

}

type IndexFileSetReader interface {

	io.Closer

	Open(opts IndexReaderOpenOptions) error

	

	// ReadSegmentFileSet returns the next segment file set or an error.

	// It will return io.EOF error when no more file sets remain.

	// The IndexSegmentFileSet will only be valid until the next read

	// or until the reader is closed if no consequent read is made.

	ReadSegmentFileSet() (IndexSegmentFileSet, error)

	Validate() error

}
```

### Bootstrap

The bootstrap process that is shared between namespaces will need to be prepared so that new instances of the bootstrapper and the sources can be made and used between the different bootstrap data and bootstrap index calls.

Here are the corresponding interfaces:

```
type RunOptions interface {

	SetIncremental(value bool) RunOptions

	Incremental() bool

}

type ProcessProvider interface {

	SetBootstrapperProvider(bootstrapper BootstrapperProvider)

	BootstrapperProvider() BootstrapperProvider

	Provide() Process

}

type Process interface {

	Run(

		ns namespace.Metadata,

		shards []uint32,

	) (ProcessResult, error)

}

type ProcessResult struct {

	DataResult result.DataBootstrapResult

	IndexResult result.IndexBootstrapResult

}

type BootstrapperProvider interface {

	String() string

	Can(strategy Strategy) bool

	Provide() Bootstrapper

}

type Bootstrapper interface {

	BootstrapData(

		ns namespace.Metadata,

		shardsTimeRanges result.ShardTimeRanges,

		opts RunOptions,

	) (result.DataBootstrapResult, error)

	BootstrapIndex(

		ns namespace.Metadata,

		shardsTimeRanges result.ShardTimeRanges,

		opts RunOptions,

	) (result.IndexBootstrapResult, error)

}

type Source interface {

	AvailableData(

		ns namespace.Metadata,

		shardsTimeRanges result.ShardTimeRanges,

	) result.ShardTimeRanges

	ReadData(

		ns namespace.Metadata,

		shardsTimeRanges result.ShardTimeRanges,

		opts RunOptions,

	) (result.DataBootstrapResult, error)

AvailableIndex(

		ns namespace.Metadata,

		shardsTimeRanges result.ShardTimeRanges,

	) result.ShardTimeRanges

	ReadIndex(

		ns namespace.Metadata,

		shardsTimeRanges result.ShardTimeRanges,

		opts RunOptions,

	) (result.IndexBootstrapResult, error)

}

// result package changes

type IndexBootstrapResult interface {

	// Blocks returns a map of all index block results.

	IndexResults() IndexResults

	// Unfulfilled is the unfulfilled time ranges for the bootstrap.

	Unfulfilled() result.ShardTimeRanges

	// SetUnfulfilled sets the current unfulfilled shard time ranges.

	SetUnfulfilled(unfulfilled result.ShardTimeRanges)

	// Add adds an index block result.

	Add(block IndexBlock, unfulfilled result.ShardTimeRanges)

}

// IndexResults is a set of index blocks indexed by block start.

type IndexResults map[xtime.UnixNano]IndexBlock

type IndexBlock struct {

	blockStart time.Time

	segments []segment.Segment

}
```

## Open issues (if applicable)

* Namespace metadata needs to have index metadata so that bootstrappers can access this per namespace

    * Also operators can then define the index metadata similar to the namespace metadata

