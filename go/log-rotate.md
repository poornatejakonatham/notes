# How can I log in golang to a file with log rotation?

I am trying to write a web application which will run on a remote server. I need to log to capture errors/debug/audit. I find that multiple logging packages are available for golang including the standard "log" package. However, I need to fulfill three requirements:

- The log files need to be rotated
- It applies to the included packages which use "log"
- It needs to be cross-platform. Dev environment is Linux and needs to be deployed on Windows.

## Answer

The best way to fulfill all your three requirements instead of creating an alternate `logger` struct, if you were satisfied using the base-level `log.Log`, is to set the output of the `logger` to your own `io.Writer` instance.

So basically what I'm going to do here is show an example where I create my own `io.Writer`:

```go
import (
    "os"
    "sync"
    "time"
)

type RotateWriter struct {
    lock     sync.Mutex
    filename string // should be set to the actual filename
    fp       *os.File
}

// Make a new RotateWriter. Return nil if error occurs during setup.
func New(filename string) *RotateWriter {
    w := &RotateWriter{filename: filename}
    err := w.Rotate()
    if err != nil {
        return nil
    }
    return w
}

// Write satisfies the io.Writer interface.
func (w *RotateWriter) Write(output []byte) (int, error) {
    w.lock.Lock()
    defer w.lock.Unlock()
    return w.fp.Write(output)
}

// Perform the actual act of rotating and reopening file.
func (w *RotateWriter) Rotate() (err error) {
    w.lock.Lock()
    defer w.lock.Unlock()

    // Close existing file if open
    if w.fp != nil {
        err = w.fp.Close()
        w.fp = nil
        if err != nil {
            return
        }
    }
    // Rename dest file if it already exists
    _, err = os.Stat(w.filename)
    if err == nil {
        err = os.Rename(w.filename, w.filename+"."+time.Now().Format(time.RFC3339))
        if err != nil {
            return
        }
    }

    // Create a file.
    w.fp, err = os.Create(w.filename)
    return
}
```

You then create a `RotateWriter` and use `log.SetOutput` to set this writer (if other packages are using the standard logger instance) or alternately create your own instances using `log.New` to pass around.

I haven't solved the situation of when to call `Rotate`, I'll leave that to you to decide. It'd be fairly simple to trigger it based on time, or alternately do so after some amount of writes or some amount of bytes.
