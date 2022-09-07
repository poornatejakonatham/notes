# Serving Single-Page Apps From Go

## Why you might care about this

You might care if you want to distribute your Go backend and single-page app (SPA) frontend together. This might be adding both to a single Docker image, or embedding your frontend into your Go binary.

## TL;DR

For those just looking for the code, you can find a sample implementation here: https://gitlab.com/hackandsla.sh/blog-examples/-/blob/master/2021-11-06-serve-spa-from-go/main.go

## Background

Single-page apps are a common way to create web applications. Most modern frameworks like Vue and React come with their own routers (e.g. vue-router and react-router) that handle frontend HTTP routes themselves. Rather than using the classical HTTP model where the client makes a request for a specific resource, SPAs expect the server to return just the root `index.html` file contents. Once the JavaScript for that file is loaded, it handles all the routing internally without needing to make subsequent requests to the server.

In the classical HTTP model, the server is simply expected to respond with the appropriate resource at a certain path. If no resource exists, a 404 error is generated:

![The classical HTTP model image](./classical-http.png "The classical HTTP model")

However in the SPA model, the server isn’t responsible for the frontend routes. So rather than generating a 404 error, we expect it to respond with the contents of the root `/index.html`:

![The SPA model image](./spa.png "The SPA model")

## Serving SPAs using Go’s `net/http` package

`net/http` has a function called `http.FileServer` which will serve the contents of the `fs.FS` given to it. For example:

```go
var myFS fs.FS = os.DirFS("mydir")
fileServer := http.FileServer(http.FS(myFS))
```

However, when the client requests a file that doesn’t exist, the `http.FileServer` responds with a 404 error. To correctly handle an SPA frontend, we need it to respond with the contents of `/index.html`. Unfortunately, `http.FileServer` is not configurable; there’s no way to alter its default behavior of returning files from the requested paths.

But we can wrap it with a small middleware that knows how to deal with 404 errors. We can do this by hooking into the `http.ResponseWriter`. Since it’s just an interface, we can implement our own version that detects when the `http.FileServer` writes a 404, and then call a different function instead.

First, we create the hooked `http.ResponseWriter`:

```go
type hookedResponseWriter struct {
    http.ResponseWriter
    got404 bool
}

func (hrw *hookedResponseWriter) WriteHeader(status int) {
    if status == http.StatusNotFound {
        // Don't actually write the 404 header, just set a flag.
        hrw.got404 = true
    } else {
        hrw.ResponseWriter.WriteHeader(status)
    }
}

func (hrw *hookedResponseWriter) Write(p []byte) (int, error) {
    if hrw.got404 {
        // No-op, but pretend that we wrote len(p) bytes to the writer.
        return len(p), nil
    }

    return hrw.ResponseWriter.Write(p)
}
```

Our response writer embeds a standard `http.ResponseWriter`, but overrides the `Write` and `WriteHeader` to set a flag if a 404 is detected.

Next we’ll make an interceptor middleware which calls a given `http.Handler` using our hooked response writer. After the handler is called, we check to see if we caught a 404. If so, it calls a secondary `http.Handler`:

```go
func intercept404(handler, on404 http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        hookedWriter := &hookedResponseWriter{ResponseWriter: w}
        handler.ServeHTTP(hookedWriter, r)

        if hookedWriter.got404 {
            on404.ServeHTTP(w, r)
        }
    })
}
```

Next, we need a way to serve the contents of our root `index.html` file. For this, we can use the `http.ServeContent` function, which serves a single file’s contents:

```go
func serveFileContents(file string, files http.FileSystem) http.HandlerFunc {
  return func(w http.ResponseWriter, r *http.Request) {
        // Restrict only to instances where the browser is looking for an HTML file
        if !strings.Contains(r.Header.Get("Accept"), "text/html") {
            w.WriteHeader(http.StatusNotFound)
            fmt.Fprint(w, "404 not found")

            return
        }

        // Open the file and return its contents using http.ServeContent
        index, err := files.Open(file)
        if err != nil {
            w.WriteHeader(http.StatusNotFound)
            fmt.Fprintf(w, "%s not found", file)

            return
        }

        fi, err := index.Stat()
        if err != nil {
            w.WriteHeader(http.StatusNotFound)
            fmt.Fprintf(w, "%s not found", file)

            return
        }

        w.Header().Set("Content-Type", "text/html; charset=utf-8")
        http.ServeContent(w, r, fi.Name(), fi.ModTime(), index)
    }
}
```

Finally, we just need to tie everything together:

```go
var frontend fs.FS = os.DirFS("frontend/dist")
httpFS := http.FS(frontend)
fileServer := http.FileServer(httpFS)
serveIndex := serveFileContents("index.html", httpFS)

http.Handle("/", intercept404(fileServer, serveIndex))
```
