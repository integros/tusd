# Integros tusd

<img alt="Tus logo" src="https://github.com/tus/tus.io/blob/master/assets/img/tus1.png?raw=true" width="30%" align="right" />

> **tus** is a protocol based on HTTP for *resumable file uploads*. Resumable
> means that an upload can be interrupted at any moment and can be resumed without
> re-uploading the previous data again. An interruption may happen willingly, if
> the user wants to pause, or by accident in case of an network issue or server
> outage.

tusd is the official reference implementation of the [tus resumable upload
protocol](http://www.tus.io/protocols/resumable-upload.html). The protocol
specifies a flexible method to upload files to remote servers using HTTP.
The special feature is the ability to pause and resume uploads at any
moment allowing to continue seamlessly after e.g. network interruptions.

**Protocol version:** 1.0.0

## Getting started

### Download pre-builts binaries (recommended)

You can download ready-to-use packages including binaries for OS X, Linux and
Windows in various formats of the
[latest release](https://github.com/tus/tusd/releases/latest).

### Compile from source

**Requirements:**

* [Go](http://golang.org/doc/install) (1.5 or newer)

**Running tusd from source:**

Clone the git repository and `cd` into it.

```bash
git clone git@github.com:tus/tusd.git
cd tusd
```

Now you can run tusd:

```bash
go run cmd/tusd/main.go
```

## Using tusd manually

Besides from running tusd using the provided binary, you can embed it into
your own Go program:

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/tus/tusd"
	"github.com/tus/tusd/filestore"
)

func main() {
	// Create a new FileStore instance which is responsible for
	// storing the uploaded file on disk in the specified directory.
	// This path _must_ exist before tusd will store uploads in it.
	// If you want to save them on a different medium, for example
	// a remote FTP server, you can implement your own storage backend
	// by implementing the tusd.DataStore interface.
	store := filestore.FileStore{
		Path: "./uploads",
	}

	// A storage backend for tusd may consist of multiple different parts which
	// handle upload creation, locking, termination and so on. The composer is a
	// place where all those seperated pieces are joined together. In this example
	// we only use the file store but you may plug in multiple.
	composer := tusd.NewStoreComposer()
	store.UseIn(composer)

	// Create a new HTTP handler for the tusd server by providing a configuration.
	// The StoreComposer property must be set to allow the handler to function.
	handler, err := tusd.NewHandler(tusd.Config{
		BasePath:      "/files/",
		StoreComposer: composer,
	})
	if err != nil {
		panic(fmt.Errorf("Unable to create handler: %s", err))
	}

	// Right now, nothing has happened since we need to start the HTTP server on
	// our own. In the end, tusd will start listening on and accept request at
	// http://localhost:8080/files
	http.Handle("/files/", http.StripPrefix("/files/", handler))
	err = http.ListenAndServe(":8080", nil)
	if err != nil {
		panic(fmt.Errorf("Unable to listen: %s", err))
	}
}

```

Please consult the [online documentation](https://godoc.org/github.com/tus/tusd)
for more details about tusd's APIs and its sub-packages.

## Implementing own storages

The tusd server is built to be as flexible as possible and to allow the use
of different upload storage mechanisms. By default the tusd binary includes
[`filestore`](https://godoc.org/github.com/tus/tusd/filestore) which will save every upload
to a specific directory on disk.

If you have different requirements, you can build your own storage backend
which will save the files to S3, a remote FTP server or similar. Doing so
is as simple as implementing the [`tusd.DataStore`](https://godoc.org/github.com/tus/tusd/#DataStore)
interface and using the new struct in the [configuration object](https://godoc.org/github.com/tus/tusd/#Config).
Please consult the documentation about detailed information about the
required methods.

## Packages

This repository does not only contain the HTTP server's code but also other
useful tools:

* [**s3store**](https://godoc.org/github.com/tus/tusd/s3store): A storage backend using AWS S3
* [**filestore**](https://godoc.org/github.com/tus/tusd/filestore): A storage backend using the local file system
* [**memorylocker**](https://godoc.org/github.com/tus/tusd/memorylocker): An in-memory locker for handling concurrent uploads
* [**consullocker**](https://godoc.org/github.com/tus/tusd/consullocker): A locker using the distributed Consul service
* [**limitedstore**](https://godoc.org/github.com/tus/tusd/limitedstore): A storage wrapper limiting the total used space for uploads

## Running the testsuite

[![Build Status](https://travis-ci.org/tus/tusd.svg?branch=master)](https://travis-ci.org/tus/tusd)
[![Build status](https://ci.appveyor.com/api/projects/status/2y6fa4nyknoxmyc8/branch/master?svg=true)](https://ci.appveyor.com/project/Acconut/tusd/branch/master)

```bash
go test -v ./...
```

## FAQ

### How can I access tusd using HTTPS?

The tusd binary, once executed, listens on the provided port for only non-encrypted HTTP requests and *does not accept* HTTPS connections. This decision has been made to limit the functionality inside this repository which has to be developed, tested and maintained. If you want to send requests to tusd in a secure fashion - what we absolutely encourage, we recommend you to utilize a reverse proxy in front of tusd which accepts incoming HTTPS connections and forwards them to tusd using plain HTTP. More information about this topic, including sample configurations for Nginx and Apache, can be found in [issue #86](https://github.com/tus/tusd/issues/86#issuecomment-269569077) and in the [Apache example configuration](.infra/files/apache2.conf).

### Can I run tusd behind a reverse proxy?

Yes, it is absolutely possible to do so. Firstly, you should execute the tusd binary using the `-behind-proxy` flag indicating it to pay attention to special headers which are only relevent when used in conjunction with a proxy. Furthermore, there are addtional details which should be kept in mind, depending on the used software:

- *Disable request buffering.* Nginx, for example, reads the entire incoming HTTP request, including its body, before sending it to the backend, by default. This behavior defeats the purpose of resumability where an upload is processed while it's being transfered. Therefore, such as feature should be disabled.

- *Adjust maximum request size.* Some proxies have default values for how big a request may be in order to protect your services. Be sure to check these settings to match the requirements of your application.

- *Forward hostname and scheme.* If the proxy rewrites the request URL, the tusd server does not know the original URL which was used to reach the proxy. This behavior can lead to situations, where tusd returns a redirect to a URL which can not be reached by the client. To avoid this confusion, you can explicitly tell tusd which hostname and scheme to use by supplying the `X-Forwarded-Host` and `X-Forwarded-Proto` headers.

Explicit examples for the above points can be found in the [Nginx configuration](https://github.com/tus/tusd/blob/master/.infra/files/nginx.conf) which is used to power the [master.tus.io](https://master.tus.io) instace.

### Can I run custom verification/authentication checks before an upload begins?

Yes, this is made possible by the [hook system](/docs/hooks.md) inside the tusd binary. It enables custom routines to be executed when certain events occurs, such as a new upload being created which can be handled by the `pre-create` hook. Inside the corresponding hook file, you can run your own validations against the provided upload metadata to determine whether the action is actually allowed or should be rejected by tusd. Please have a look at the [corresponding documentation](docs/hooks.md#pre-create) for a more detailed explanation.


## License

This project is licensed under the MIT license, see `LICENSE.txt`.
