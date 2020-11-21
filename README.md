# request
[![PkgGoDev](https://pkg.go.dev/badge/github.com/hslam/request)](https://pkg.go.dev/github.com/hslam/request)
[![Build Status](https://travis-ci.org/hslam/request.svg?branch=master)](https://travis-ci.org/hslam/request)
[![Go Report Card](https://goreportcard.com/badge/github.com/hslam/request)](https://goreportcard.com/report/github.com/hslam/request)
[![LICENSE](https://img.shields.io/github/license/hslam/request.svg?style=flat-square)](https://github.com/hslam/request/blob/master/LICENSE)

Package request implements an HTTP request reader.

## Get started

### Install
```
go get github.com/hslam/request
```
### Import
```
import "github.com/hslam/request"
```
### Usage
#### Example
```go
package main

import (
	"bufio"
	"github.com/hslam/mux"
	"github.com/hslam/request"
	"github.com/hslam/response"
	"net"
	"net/http"
)

func main() {
	m := mux.New()
	m.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Hello World"))
	})
	ListenAndServe(":8080", m)
}

func ListenAndServe(addr string, handler http.Handler) error {
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	for {
		conn, err := ln.Accept()
		if err != nil {
			return err
		}
		go func(conn net.Conn) {
			reader := bufio.NewReader(conn)
			rw := bufio.NewReadWriter(reader, bufio.NewWriter(conn))
			var err error
			var req *http.Request
			for {
				req, err = request.ReadFastRequest(reader)
				if err != nil {
					break
				}
				res := response.NewResponse(req, conn, rw)
				handler.ServeHTTP(res, req)
				res.FinishRequest()
				request.FreeRequest(req)
				response.FreeResponse(res)
			}
		}(conn)
	}
}
```

#### Netpoll Example
```go
package main

import (
	"bufio"
	"github.com/hslam/mux"
	"github.com/hslam/netpoll"
	"github.com/hslam/request"
	"github.com/hslam/response"
	"net"
	"net/http"
	"sync"
)

func main() {
	m := mux.New()
	m.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Hello World"))
	})
	ListenAndServe(":8080", m)
}

func ListenAndServe(addr string, handler http.Handler) error {
	var h = &netpoll.ConnHandler{}
	type Context struct {
		reader  *bufio.Reader
		rw      *bufio.ReadWriter
		conn    net.Conn
		serving sync.Mutex
	}
	h.SetUpgrade(func(conn net.Conn) (netpoll.Context, error) {
		reader := bufio.NewReader(conn)
		rw := bufio.NewReadWriter(reader, bufio.NewWriter(conn))
		return &Context{reader: reader, conn: conn, rw: rw}, nil
	})
	h.SetServe(func(context netpoll.Context) error {
		ctx := context.(*Context)
		ctx.serving.Lock()
		req, err := request.ReadFastRequest(ctx.reader)
		if err != nil {
			ctx.serving.Unlock()
			return err
		}
		res := response.NewResponse(req, ctx.conn, ctx.rw)
		handler.ServeHTTP(res, req)
		res.FinishRequest()
		ctx.serving.Unlock()
		request.FreeRequest(req)
		response.FreeResponse(res)
		return nil
	})
	return netpoll.ListenAndServe("tcp", addr, h)
}
```

curl -XGET http://localhost:8080
```
Hello World
```

### License
This package is licensed under a MIT license (Copyright (c) 2020 Meng Huang)


### Author
request was written by Meng Huang.


