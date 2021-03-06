# Separation of cmdlogbuf module into two modules

> [Merged PR: ARCUS-Persistence](https://github.com/naver/arcus-memcached/pull/509)

- **cmdlogbuf** 모듈을 기능에 따라 두 개의 모듈로 분리
    1. Log buffer 및 log flusher 모듈: `cmdlogbuf.c`, `cmdlogbuf.h`
    2. Log file 모듈: `cmdlogfile.c`, `cmdlogfile.h`
- 기존 `cmdlogbuf.c` 내의 data structure, function 등을 분리해 아래 정리

## Log Buffer

```cpp
/* FIXME: config log buffer size */
#define CMDLOG_BUFFER_SIZE (100 * 1024 * 1024) /* 100 MB */
#define CMDLOG_FLUSH_AUTO_SIZE (32 * 1024) /* 32 KB : see the nflush data type of log_FREQ */
#define CMDLOG_RECORD_MIN_SIZE 16          /* 8 bytes header + 8 bytes body */

static EXTENSION_LOGGER_DESCRIPTOR* logger = NULL;
static struct log_global log_gl;

/* flush request structure */
typedef struct _log_freq {
    uint16_t  nflush;     /* amount of log buffer to flush */
    uint8_t   dual_write; /* flag of dual write */
    uint8_t   unused;
} log_FREQ;

/* log buffer structure */
typedef struct _log_buffer {
    /* log buffer */
    char       *data;   /* log buffer pointer */
    uint32_t    size;   /* log buffer size */
    uint32_t    head;   /* the head position in log buffer */
    uint32_t    tail;   /* the tail position in log buffer */
    int32_t     last;   /* the last position in log buffer */

    /* flush request queue */
    log_FREQ   *fque;   /* flush request queue pointer */
    uint32_t    fqsz;   /* flush request queue size */
    uint32_t    fbgn;   /* the queue index to begin flush */
    uint32_t    fend;   /* the queue index to end flush */
    int32_t     dw_end; /* the queue index to end dual write */
} log_BUFFER;

/* log flusher structure */
typedef struct _log_flusher {
    pthread_mutex_t  lock;
    pthread_cond_t   cond;
    bool             sleep;
    volatile uint8_t running;
    volatile bool    reqstop;
} log_FLUSHER;

/* log global structure */
struct log_global {
    log_FILE        log_file; // file
    log_BUFFER      log_buffer;
    log_FLUSHER     log_flusher;
    LogSN           nxt_write_lsn;
    LogSN           nxt_flush_lsn;
    LogSN           nxt_fsync_lsn; // file
    pthread_mutex_t log_write_lock; // cmdlog_file_getsize
    pthread_mutex_t log_flush_lock; // buff+file
    pthread_mutex_t log_fsync_lock; // file: except buff_init
    pthread_mutex_t flush_lsn_lock;
    pthread_mutex_t fsync_lsn_lock; // file: except buff_init
    pthread_cond_t  log_flush_cond; // file: except buff_init
    bool            async_mode; // file: except buff_init
    volatile bool   initialized; // file: except buff_init/cmdlog_buf_flush_thread_start
};

static void do_log_flusher_wakeup(log_FLUSHER *flusher)
static uint32_t do_log_buff_flush(bool flush_all)
static LogSN do_log_buff_write(LogRec *logrec, bool dual_write)
static void do_log_buff_complete_dual_write(bool success)

static void *log_flush_thread_main(void *arg)

void cmdlog_buff_write(LogRec *logrec, log_waiter_t *waiter, bool dual_write)
void cmdlog_buff_flush(LogSN *upto_lsn)

void cmdlog_get_flush_lsn(LogSN *lsn)
void cmdlog_get_fsync_lsn(LogSN *lsn)

void cmdlog_complete_dual_write(bool success)

ENGINE_ERROR_CODE cmdlog_buf_init(struct default_engine* engine)
void cmdlog_buf_final(void)

ENGINE_ERROR_CODE cmdlog_buf_flush_thread_start(void)
void cmdlog_buf_flush_thread_stop(void)
```

## Log File

```cpp
#define CMDLOG_MAX_FILEPATH_LENGTH 255

/* log file structure */
typedef struct _log_file {
    char      path[CMDLOG_MAX_FILEPATH_LENGTH+1];
    int       prev_fd;
    int       fd;
    int       next_fd;
    size_t    size;
    size_t    next_size;
    bool      close_wait;
} log_FILE;

static EXTENSION_LOGGER_DESCRIPTOR* logger = NULL;

static int disk_open(const char *fname, int flags, int mode)
static off_t disk_lseek(int fd, off_t offset, int whence)
static ssize_t disk_read(int fd, void *buf, size_t count)
static ssize_t disk_write(int fd, void *buf, size_t count)
static int disk_fsync(int fd)
static int disk_close(int fd)

static void do_log_file_close_waiter_wakeup(log_FILE *file)
static void do_log_file_write(char *log_ptr, uint32_t log_size, bool dual_write)
static void do_log_file_complete_dual_write(void)

void cmdlog_file_sync(void)

int cmdlog_file_open(char *path)
void cmdlog_file_close(bool chkpt_success)
void cmdlog_file_init(void)
void cmdlog_file_final(void)
size_t cmdlog_file_getsize(void)

static int do_redo_pending_lrec(int fd, int start_offset, int end_offset)

int cmdlog_file_apply(void)
```