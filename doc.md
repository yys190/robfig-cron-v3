### Overview [¶](#pkg-overview "Go to Overview")

+   [Installation](#hdr-Installation)
+   [Usage](#hdr-Usage)
+   [CRON Expression Format](#hdr-CRON_Expression_Format)
+   [Alternative Formats](#hdr-Alternative_Formats)
+   [Special Characters](#hdr-Special_Characters)
+   [Predefined schedules](#hdr-Predefined_schedules)
+   [Intervals](#hdr-Intervals)
+   [Time zones](#hdr-Time_zones)
+   [Job Wrappers](#hdr-Job_Wrappers)
+   [Thread safety](#hdr-Thread_safety)
+   [Logging](#hdr-Logging)
+   [Implementation](#hdr-Implementation)

Package cron implements a cron spec parser and job runner.

#### Installation [¶](#hdr-Installation "Go to Installation")

To download the specific tagged release, run:

```
go get github.com/robfig/cron/v3@v3.0.0
```

Import it in your program as:

```
import "github.com/robfig/cron/v3"
```

It requires Go 1.11 or later due to usage of Go Modules.

#### Usage [¶](#hdr-Usage "Go to Usage")

Callers may register Funcs to be invoked on a given schedule. Cron will run them in their own goroutines.

```
c := cron.New()
c.AddFunc("30 * * * *", func() { fmt.Println("Every hour on the half hour") })
c.AddFunc("30 3-6,20-23 * * *", func() { fmt.Println(".. in the range 3-6am, 8-11pm") })
c.AddFunc("CRON_TZ=Asia/Tokyo 30 04 * * *", func() { fmt.Println("Runs at 04:30 Tokyo time every day") })
c.AddFunc("@hourly",      func() { fmt.Println("Every hour, starting an hour from now") })
c.AddFunc("@every 1h30m", func() { fmt.Println("Every hour thirty, starting an hour thirty from now") })
c.Start()
..
// Funcs are invoked in their own goroutine, asynchronously.
...
// Funcs may also be added to a running Cron
c.AddFunc("@daily", func() { fmt.Println("Every day") })
..
// Inspect the cron job entries' next and previous run times.
inspect(c.Entries())
..
c.Stop()  // Stop the scheduler (does not stop any jobs already running).
```

#### CRON Expression Format [¶](#hdr-CRON_Expression_Format "Go to CRON Expression Format")

A cron expression represents a set of times, using 5 space-separated fields.

```
Field name   | Mandatory? | Allowed values  | Allowed special characters
----------   | ---------- | --------------  | --------------------------
Minutes      | Yes        | 0-59            | * / , -
Hours        | Yes        | 0-23            | * / , -
Day of month | Yes        | 1-31            | * / , - ?
Month        | Yes        | 1-12 or JAN-DEC | * / , -
Day of week  | Yes        | 0-6 or SUN-SAT  | * / , - ?
```

Month and Day-of-week field values are case insensitive. "SUN", "Sun", and "sun" are equally accepted.

The specific interpretation of the format is based on the Cron Wikipedia page: [https://en.wikipedia.org/wiki/Cron](https://en.wikipedia.org/wiki/Cron)

#### Alternative Formats [¶](#hdr-Alternative_Formats "Go to Alternative Formats")

Alternative Cron expression formats support other fields like seconds. You can implement that by creating a custom Parser as follows.

```
cron.New(
	cron.WithParser(
		cron.NewParser(
			cron.SecondOptional | cron.Minute | cron.Hour | cron.Dom | cron.Month | cron.Dow | cron.Descriptor)))
```

Since adding Seconds is the most common modification to the standard cron spec, cron provides a builtin function to do that, which is equivalent to the custom parser you saw earlier, except that its seconds field is REQUIRED:

```
cron.New(cron.WithSeconds())
```

That emulates Quartz, the most popular alternative Cron schedule format: [http://www.quartz-scheduler.org/documentation/quartz-2.x/tutorials/crontrigger.html](http://www.quartz-scheduler.org/documentation/quartz-2.x/tutorials/crontrigger.html)

#### Special Characters [¶](#hdr-Special_Characters "Go to Special Characters")

Asterisk ( \* )

The asterisk indicates that the cron expression will match for all values of the field; e.g., using an asterisk in the 5th field (month) would indicate every month.

Slash ( / )

Slashes are used to describe increments of ranges. For example 3-59/15 in the 1st field (minutes) would indicate the 3rd minute of the hour and every 15 minutes thereafter. The form "\*\\/..." is equivalent to the form "first-last/...", that is, an increment over the largest possible range of the field. The form "N/..." is accepted as meaning "N-MAX/...", that is, starting at N, use the increment until the end of that specific range. It does not wrap around.

Comma ( , )

Commas are used to separate items of a list. For example, using "MON,WED,FRI" in the 5th field (day of week) would mean Mondays, Wednesdays and Fridays.

Hyphen ( - )

Hyphens are used to define ranges. For example, 9-17 would indicate every hour between 9am and 5pm inclusive.

Question mark ( ? )

Question mark may be used instead of '\*' for leaving either day-of-month or day-of-week blank.

#### Predefined schedules [¶](#hdr-Predefined_schedules "Go to Predefined schedules")

You may use one of several pre-defined schedules in place of a cron expression.

```
Entry                  | Description                                | Equivalent To
-----                  | -----------                                | -------------
@yearly (or @annually) | Run once a year, midnight, Jan. 1st        | 0 0 1 1 *
@monthly               | Run once a month, midnight, first of month | 0 0 1 * *
@weekly                | Run once a week, midnight between Sat/Sun  | 0 0 * * 0
@daily (or @midnight)  | Run once a day, midnight                   | 0 0 * * *
@hourly                | Run once an hour, beginning of hour        | 0 * * * *
```

#### Intervals [¶](#hdr-Intervals "Go to Intervals")

You may also schedule a job to execute at fixed intervals, starting at the time it's added or cron is run. This is supported by formatting the cron spec like this:

```
@every <duration>
```

where "duration" is a string accepted by time.ParseDuration ([http://golang.org/pkg/time/#ParseDuration](http://golang.org/pkg/time/#ParseDuration)).

For example, "@every 1h30m10s" would indicate a schedule that activates after 1 hour, 30 minutes, 10 seconds, and then every interval after that.

Note: The interval does not take the job runtime into account. For example, if a job takes 3 minutes to run, and it is scheduled to run every 5 minutes, it will have only 2 minutes of idle time between each run.

#### Time zones [¶](#hdr-Time_zones "Go to Time zones")

By default, all interpretation and scheduling is done in the machine's local time zone (time.Local). You can specify a different time zone on construction:

```
cron.New(
    cron.WithLocation(time.UTC))
```

Individual cron schedules may also override the time zone they are to be interpreted in by providing an additional space-separated field at the beginning of the cron spec, of the form "CRON\_TZ=Asia/Tokyo".

For example:

```
# Runs at 6am in time.Local
cron.New().AddFunc("0 6 * * ?", ...)

# Runs at 6am in America/New_York
nyc, _ := time.LoadLocation("America/New_York")
c := cron.New(cron.WithLocation(nyc))
c.AddFunc("0 6 * * ?", ...)

# Runs at 6am in Asia/Tokyo
cron.New().AddFunc("CRON_TZ=Asia/Tokyo 0 6 * * ?", ...)

# Runs at 6am in Asia/Tokyo
c := cron.New(cron.WithLocation(nyc))
c.SetLocation("America/New_York")
c.AddFunc("CRON_TZ=Asia/Tokyo 0 6 * * ?", ...)
```

The prefix "TZ=(TIME ZONE)" is also supported for legacy compatibility.

Be aware that jobs scheduled during daylight-savings leap-ahead transitions will not be run!

#### Job Wrappers [¶](#hdr-Job_Wrappers "Go to Job Wrappers")

A Cron runner may be configured with a chain of job wrappers to add cross-cutting functionality to all submitted jobs. For example, they may be used to achieve the following effects:

+   Recover any panics from jobs (activated by default)
+   Delay a job's execution if the previous run hasn't completed yet
+   Skip a job's execution if the previous run hasn't completed yet
+   Log each job's invocations

Install wrappers for all jobs added to a cron using the \`cron.WithChain\` option:

```
cron.New(cron.WithChain(
	cron.SkipIfStillRunning(logger),
))
```

Install wrappers for individual jobs by explicitly wrapping them:

```
job = cron.NewChain(
	cron.SkipIfStillRunning(logger),
).Then(job)
```

#### Thread safety [¶](#hdr-Thread_safety "Go to Thread safety")

Since the Cron service runs concurrently with the calling code, some amount of care must be taken to ensure proper synchronization.

All cron methods are designed to be correctly synchronized as long as the caller ensures that invocations have a clear happens-before ordering between them.

#### Logging [¶](#hdr-Logging "Go to Logging")

Cron defines a Logger interface that is a subset of the one defined in github.com/go-logr/logr. It has two logging levels (Info and Error), and parameters are key/value pairs. This makes it possible for cron logging to plug into structured logging systems. An adapter, \[Verbose\]PrintfLogger, is provided to wrap the standard library \*log.Logger.

For additional insight into Cron operations, verbose logging may be activated which will record job runs, scheduling decisions, and added or removed jobs. Activate it with a one-off logger as follows:

```
cron.New(
	cron.WithLogger(
		cron.VerbosePrintfLogger(log.New(os.Stdout, "cron: ", log.LstdFlags))))
```

#### Implementation [¶](#hdr-Implementation "Go to Implementation")

Cron entries are stored in an array, sorted by their next activation time. Cron sleeps until the next job is due to be run.

Upon waking:

+   it runs each entry that is active on that second
+   it calculates the next run times for the jobs that were run
+   it re-sorts the array of entries by next activation time.
+   it goes to sleep until the soonest job.

### Index [¶](#pkg-index "Go to Index")

+   [type Chain](#Chain)
+   +   [func NewChain(c ...JobWrapper) Chain](#NewChain)
+   +   [func (c Chain) Then(j Job) Job](#Chain.Then)
+   [type ConstantDelaySchedule](#ConstantDelaySchedule)
+   +   [func Every(duration time.Duration) ConstantDelaySchedule](#Every)
+   +   [func (schedule ConstantDelaySchedule) Next(t time.Time) time.Time](#ConstantDelaySchedule.Next)
+   [type Cron](#Cron)
+   +   [func New(opts ...Option) \*Cron](#New)
+   +   [func (c \*Cron) AddFunc(spec string, cmd func()) (EntryID, error)](#Cron.AddFunc)
    +   [func (c \*Cron) AddJob(spec string, cmd Job) (EntryID, error)](#Cron.AddJob)
    +   [func (c \*Cron) Entries() \[\]Entry](#Cron.Entries)
    +   [func (c \*Cron) Entry(id EntryID) Entry](#Cron.Entry)
    +   [func (c \*Cron) Location() \*time.Location](#Cron.Location)
    +   [func (c \*Cron) Remove(id EntryID)](#Cron.Remove)
    +   [func (c \*Cron) Run()](#Cron.Run)
    +   [func (c \*Cron) Schedule(schedule Schedule, cmd Job) EntryID](#Cron.Schedule)
    +   [func (c \*Cron) Start()](#Cron.Start)
    +   [func (c \*Cron) Stop() context.Context](#Cron.Stop)
+   [type Entry](#Entry)
+   +   [func (e Entry) Valid() bool](#Entry.Valid)
+   [type EntryID](#EntryID)
+   [type FuncJob](#FuncJob)
+   +   [func (f FuncJob) Run()](#FuncJob.Run)
+   [type Job](#Job)
+   [type JobWrapper](#JobWrapper)
+   +   [func DelayIfStillRunning(logger Logger) JobWrapper](#DelayIfStillRunning)
    +   [func Recover(logger Logger) JobWrapper](#Recover)
    +   [func SkipIfStillRunning(logger Logger) JobWrapper](#SkipIfStillRunning)
+   [type Logger](#Logger)
+   +   [func PrintfLogger(l interface{ ... }) Logger](#PrintfLogger)
    +   [func VerbosePrintfLogger(l interface{ ... }) Logger](#VerbosePrintfLogger)
+   [type Option](#Option)
+   +   [func WithChain(wrappers ...JobWrapper) Option](#WithChain)
    +   [func WithLocation(loc \*time.Location) Option](#WithLocation)
    +   [func WithLogger(logger Logger) Option](#WithLogger)
    +   [func WithParser(p ScheduleParser) Option](#WithParser)
    +   [func WithSeconds() Option](#WithSeconds)
+   [type ParseOption](#ParseOption)
+   [type Parser](#Parser)
+   +   [func NewParser(options ParseOption) Parser](#NewParser)
+   +   [func (p Parser) Parse(spec string) (Schedule, error)](#Parser.Parse)
+   [type Schedule](#Schedule)
+   +   [func ParseStandard(standardSpec string) (Schedule, error)](#ParseStandard)
+   [type ScheduleParser](#ScheduleParser)
+   [type SpecSchedule](#SpecSchedule)
+   +   [func (s \*SpecSchedule) Next(t time.Time) time.Time](#SpecSchedule.Next)

### Constants [¶](#pkg-constants "Go to Constants")

This section is empty.

### Variables [¶](#pkg-variables "Go to Variables")

This section is empty.

### Functions [¶](#pkg-functions "Go to Functions")

This section is empty.

### Types [¶](#pkg-types "Go to Types")

#### type [Chain](https://github.com/robfig/cron/blob/v3.0.1/chain.go#L15) [¶](#Chain "Go to Chain")

```
type Chain struct {
	// contains filtered or unexported fields
}
```

Chain is a sequence of JobWrappers that decorates submitted jobs with cross-cutting behaviors like logging or synchronization.

#### func [NewChain](https://github.com/robfig/cron/blob/v3.0.1/chain.go#L20) [¶](#NewChain "Go to NewChain")

```
func NewChain(c ...JobWrapper) Chain
```

NewChain returns a Chain consisting of the given JobWrappers.

#### func (Chain) [Then](https://github.com/robfig/cron/blob/v3.0.1/chain.go#L30) [¶](#Chain.Then "Go to Chain.Then")

```
func (c Chain) Then(j Job) Job
```

Then decorates the given job with all JobWrappers in the chain.

This:

```
NewChain(m1, m2, m3).Then(job)
```

is equivalent to:

```
m1(m2(m3(job)))
```

#### type [ConstantDelaySchedule](https://github.com/robfig/cron/blob/v3.0.1/constantdelay.go#L7) [¶](#ConstantDelaySchedule "Go to ConstantDelaySchedule")

ConstantDelaySchedule represents a simple recurring duty cycle, e.g. "Every 5 minutes". It does not support jobs more frequent than once a second.

#### func [Every](https://github.com/robfig/cron/blob/v3.0.1/constantdelay.go#L14) [¶](#Every "Go to Every")

Every returns a crontab Schedule that activates once every duration. Delays of less than a second are not supported (will round up to 1 second). Any fields less than a Second are truncated.

#### func (ConstantDelaySchedule) [Next](https://github.com/robfig/cron/blob/v3.0.1/constantdelay.go#L25) [¶](#ConstantDelaySchedule.Next "Go to ConstantDelaySchedule.Next")

Next returns the next time this should be run. This rounds so that the next activation time will be on the second.

#### type [Cron](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L13) [¶](#Cron "Go to Cron")

```
type Cron struct {
	// contains filtered or unexported fields
}
```

Cron keeps track of any number of entries, invoking the associated func as specified by the schedule. It may be started, stopped, and the entries may be inspected while running.

#### func [New](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L113) [¶](#New "Go to New")

```
func New(opts ...Option) *Cron
```

New returns a new Cron job runner, modified by the given options.

Available Settings

```
Time Zone
  Description: The time zone in which schedules are interpreted
  Default:     time.Local

Parser
  Description: Parser converts cron spec strings into cron.Schedules.
  Default:     Accepts this spec: https://en.wikipedia.org/wiki/Cron

Chain
  Description: Wrap submitted jobs to customize behavior.
  Default:     A chain that recovers panics and logs them to stderr.
```

See "cron.With\*" to modify the default behavior.

#### func (\*Cron) [AddFunc](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L141) [¶](#Cron.AddFunc "Go to Cron.AddFunc")

AddFunc adds a func to the Cron to be run on the given schedule. The spec is parsed using the time zone of this Cron instance as the default. An opaque ID is returned that can be used to later remove it.

#### func (\*Cron) [AddJob](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L148) [¶](#Cron.AddJob "Go to Cron.AddJob")

AddJob adds a Job to the Cron to be run on the given schedule. The spec is parsed using the time zone of this Cron instance as the default. An opaque ID is returned that can be used to later remove it.

#### func (\*Cron) [Entries](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L177) [¶](#Cron.Entries "Go to Cron.Entries")

```
func (c *Cron) Entries() []Entry
```

Entries returns a snapshot of the cron entries.

#### func (\*Cron) [Entry](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L194) [¶](#Cron.Entry "Go to Cron.Entry")

```
func (c *Cron) Entry(id EntryID) Entry
```

Entry returns a snapshot of the given entry, or nil if it couldn't be found.

#### func (\*Cron) [Location](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L189) [¶](#Cron.Location "Go to Cron.Location")

Location gets the time zone location

#### func (\*Cron) [Remove](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L204) [¶](#Cron.Remove "Go to Cron.Remove")

```
func (c *Cron) Remove(id EntryID)
```

Remove an entry from being run in the future.

#### func (\*Cron) [Run](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L226) [¶](#Cron.Run "Go to Cron.Run")

Run the cron scheduler, or no-op if already running.

#### func (\*Cron) [Schedule](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L158) [¶](#Cron.Schedule "Go to Cron.Schedule")

```
func (c *Cron) Schedule(schedule Schedule, cmd Job) EntryID
```

Schedule adds a Job to the Cron to be run on the given schedule. The job is wrapped with the configured Chain.

#### func (\*Cron) [Start](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L215) [¶](#Cron.Start "Go to Cron.Start")

Start the cron scheduler in its own goroutine, or no-op if already started.

#### func (\*Cron) [Stop](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L323) [¶](#Cron.Stop "Go to Cron.Stop")

Stop stops the cron scheduler if it is running; otherwise it does nothing. A context is returned so the caller can wait for running jobs to complete.

#### type [Entry](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L50) [¶](#Entry "Go to Entry")

```
type Entry struct {
	// ID is the cron-assigned ID of this entry, which may be used to look up a
	// snapshot or remove it.
	ID EntryID

	// Schedule on which this job should be run.
	Schedule Schedule

	// Next time the job will run, or the zero time if Cron has not been
	// started or this entry's schedule is unsatisfiable
	Next time.Time

	// Prev is the last time this job was run, or the zero time if never.
	Prev time.Time

	// WrappedJob is the thing to run when the Schedule is activated.
	WrappedJob Job

	// Job is the thing that was submitted to cron.
	// It is kept around so that user code that needs to get at the job later,
	// e.g. via Entries() can do so.
	Job Job
}
```

Entry consists of a schedule and the func to execute on that schedule.

#### func (Entry) [Valid](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L75) [¶](#Entry.Valid "Go to Entry.Valid")

Valid returns true if this is not the zero entry.

#### type [EntryID](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L47) [¶](#EntryID "Go to EntryID")

EntryID identifies an entry within a Cron instance

#### type [FuncJob](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L134) [¶](#FuncJob "Go to FuncJob")

FuncJob is a wrapper that turns a func() into a cron.Job

#### func (FuncJob) [Run](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L136) [¶](#FuncJob.Run "Go to FuncJob.Run")

#### type [Job](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L35) [¶](#Job "Go to Job")

```
type Job interface {
	Run()
}
```

Job is an interface for submitted cron jobs.

#### type [JobWrapper](https://github.com/robfig/cron/blob/v3.0.1/chain.go#L11) [¶](#JobWrapper "Go to JobWrapper")

```
type JobWrapper func(Job) Job
```

JobWrapper decorates the given Job with some behavior.

#### func [DelayIfStillRunning](https://github.com/robfig/cron/blob/v3.0.1/chain.go#L61) [¶](#DelayIfStillRunning "Go to DelayIfStillRunning")

```
func DelayIfStillRunning(logger Logger) JobWrapper
```

DelayIfStillRunning serializes jobs, delaying subsequent runs until the previous one is complete. Jobs running after a delay of more than a minute have the delay logged at Info.

#### func [Recover](https://github.com/robfig/cron/blob/v3.0.1/chain.go#L38) [¶](#Recover "Go to Recover")

```
func Recover(logger Logger) JobWrapper
```

Recover panics in wrapped jobs and log them with the provided logger.

#### func [SkipIfStillRunning](https://github.com/robfig/cron/blob/v3.0.1/chain.go#L78) [¶](#SkipIfStillRunning "Go to SkipIfStillRunning")

```
func SkipIfStillRunning(logger Logger) JobWrapper
```

SkipIfStillRunning skips an invocation of the Job if a previous invocation is still running. It logs skips to the given logger at Info level.

#### type [Logger](https://github.com/robfig/cron/blob/v3.0.1/logger.go#L19) [¶](#Logger "Go to Logger")

```
type Logger interface {
	// Info logs routine messages about cron's operation.
	Info(msg string, keysAndValues ...interface{})
	// Error logs an error condition.
	Error(err error, msg string, keysAndValues ...interface{})
}
```

Logger is the interface used in this package for logging, so that any backend can be plugged in. It is a subset of the github.com/go-logr/logr interface.

DefaultLogger is used by Cron if none is specified.

DiscardLogger can be used by callers to discard all log messages.

#### func [PrintfLogger](https://github.com/robfig/cron/blob/v3.0.1/logger.go#L28) [¶](#PrintfLogger "Go to PrintfLogger")

```
func PrintfLogger(l interface{ Printf(string, ...interface{}) }) Logger
```

PrintfLogger wraps a Printf-based logger (such as the standard library "log") into an implementation of the Logger interface which logs errors only.

#### func [VerbosePrintfLogger](https://github.com/robfig/cron/blob/v3.0.1/logger.go#L34) [¶](#VerbosePrintfLogger "Go to VerbosePrintfLogger")

```
func VerbosePrintfLogger(l interface{ Printf(string, ...interface{}) }) Logger
```

VerbosePrintfLogger wraps a Printf-based logger (such as the standard library "log") into an implementation of the Logger interface which logs everything.

#### type [Option](https://github.com/robfig/cron/blob/v3.0.1/option.go#L8) [¶](#Option "Go to Option")

Option represents a modification to the default behavior of a Cron.

#### func [WithChain](https://github.com/robfig/cron/blob/v3.0.1/option.go#L34) [¶](#WithChain "Go to WithChain")

```
func WithChain(wrappers ...JobWrapper) Option
```

WithChain specifies Job wrappers to apply to all jobs added to this cron. Refer to the Chain\* functions in this package for provided wrappers.

#### func [WithLocation](https://github.com/robfig/cron/blob/v3.0.1/option.go#L11) [¶](#WithLocation "Go to WithLocation")

WithLocation overrides the timezone of the cron instance.

#### func [WithLogger](https://github.com/robfig/cron/blob/v3.0.1/option.go#L41) [¶](#WithLogger "Go to WithLogger")

```
func WithLogger(logger Logger) Option
```

WithLogger uses the provided logger.

#### func [WithParser](https://github.com/robfig/cron/blob/v3.0.1/option.go#L26) [¶](#WithParser "Go to WithParser")

```
func WithParser(p ScheduleParser) Option
```

WithParser overrides the parser used for interpreting job schedules.

#### func [WithSeconds](https://github.com/robfig/cron/blob/v3.0.1/option.go#L19) [¶](#WithSeconds "Go to WithSeconds")

```
func WithSeconds() Option
```

WithSeconds overrides the parser used for interpreting job schedules to include a seconds field as the first one.

#### type [ParseOption](https://github.com/robfig/cron/blob/v3.0.1/parser.go#L15) [¶](#ParseOption "Go to ParseOption")

Configuration options for creating a parser. Most options specify which fields should be included, while others enable features. If a field is not included the parser will assume a default value. These options do not change the order fields are parse in.

```
const (
	Second         ParseOption = 1 << iota // Seconds field, default 0
	SecondOptional                         // Optional seconds field, default 0
	Minute                                 // Minutes field, default 0
	Hour                                   // Hours field, default 0
	Dom                                    // Day of month field, default *
	Month                                  // Month field, default *
	Dow                                    // Day of week field, default *
	DowOptional                            // Optional day of week field, default *
	Descriptor                             // Allow descriptors such as @monthly, @weekly, etc.
)
```

#### type [Parser](https://github.com/robfig/cron/blob/v3.0.1/parser.go#L48) [¶](#Parser "Go to Parser")

```
type Parser struct {
	// contains filtered or unexported fields
}
```

A custom Parser that can be configured.

#### func [NewParser](https://github.com/robfig/cron/blob/v3.0.1/parser.go#L71) [¶](#NewParser "Go to NewParser")

```
func NewParser(options ParseOption) Parser
```

NewParser creates a Parser with custom options.

It panics if more than one Optional is given, since it would be impossible to correctly infer which optional is provided or missing in general.

Examples

```
// Standard parser without descriptors
specParser := NewParser(Minute | Hour | Dom | Month | Dow)
sched, err := specParser.Parse("0 0 15 */3 *")

// Same as above, just excludes time fields
subsParser := NewParser(Dom | Month | Dow)
sched, err := specParser.Parse("15 */3 *")

// Same as above, just makes Dow optional
subsParser := NewParser(Dom | Month | DowOptional)
sched, err := specParser.Parse("15 */3")
```

#### func (Parser) [Parse](https://github.com/robfig/cron/blob/v3.0.1/parser.go#L88) [¶](#Parser.Parse "Go to Parser.Parse")

Parse returns a new crontab schedule representing the given spec. It returns a descriptive error if the spec is not valid. It accepts crontab specs and features configured by NewParser.

#### type [Schedule](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L40) [¶](#Schedule "Go to Schedule")

```
type Schedule interface {
	// Next returns the next activation time, later than the given time.
	// Next is invoked initially, and then each time the job is run.
	Next(time.Time) time.Time
}
```

Schedule describes a job's duty cycle.

#### func [ParseStandard](https://github.com/robfig/cron/blob/v3.0.1/parser.go#L229) [¶](#ParseStandard "Go to ParseStandard")

ParseStandard returns a new crontab schedule representing the given standardSpec ([https://en.wikipedia.org/wiki/Cron](https://en.wikipedia.org/wiki/Cron)). It requires 5 entries representing: minute, hour, day of month, month and day of week, in that order. It returns a descriptive error if the spec is not valid.

It accepts

+   Standard crontab specs, e.g. "\* \* \* \* ?"
+   Descriptors, e.g. "@midnight", "@every 1h30m"

#### type [ScheduleParser](https://github.com/robfig/cron/blob/v3.0.1/cron.go#L30) [¶](#ScheduleParser "Go to ScheduleParser") added in v3.0.1

```
type ScheduleParser interface {
	Parse(spec string) (Schedule, error)
}
```

ScheduleParser is an interface for schedule spec parsers that return a Schedule

#### type [SpecSchedule](https://github.com/robfig/cron/blob/v3.0.1/spec.go#L7) [¶](#SpecSchedule "Go to SpecSchedule")

```
type SpecSchedule struct {
	Second, Minute, Hour, Dom, Month, Dow uint64

	// Override location for this schedule.
	Location *time.Location
}
```

SpecSchedule specifies a duty cycle (to the second granularity), based on a traditional crontab specification. It is computed initially and stored as bit sets.

#### func (\*SpecSchedule) [Next](https://github.com/robfig/cron/blob/v3.0.1/spec.go#L58) [¶](#SpecSchedule.Next "Go to SpecSchedule.Next")

Next returns the next time this schedule is activated, greater than the given time. If no time can be found to satisfy the schedule, return the zero time.
