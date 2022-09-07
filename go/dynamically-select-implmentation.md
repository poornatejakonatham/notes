# Dynamically selecting and running the right implementation in Golang

We have two external social media channels and want to consume their APIs. The way we would consume these APIs depends on our client's request. We have to design our application in a way that it satisfies two real life scenarios as listed below.

External APIs require different implementations when being consumed from outside. It does matter what environment consumer is using.

External APIs require exact implementations when being consumed from outside. It doesn't matter what environment consumer is using.

As you can imagine, if you are dealing with an external API that falls into the first scenario, you will highly likely have some duplications in your code or structure as well as some extra logic to deal with. However, if you are dealing with an external API that falls into the second scenario, your life would be very easy. All you would deal with is a configuration parameters.

In this example we are going to see examples for both scenarios. It hasn't finished there though. Both scenarios come with two sub scenarios as listed below.

- The client dictates the channel and the environment within the request.
- The client dictates the channel within the request but the environment is inherited from the application.

In the case of first scenario, the client sends name and env query parameters. In the case of second scenario, the client sends name query parameter. Based on the scenarios above, our application will pick up the right **channel** and **environment** then hit external API accordingly. It's a kind of *strategy design pattern*.

Finally these are our examples. We will create token, refresh token, send message and upload image as API contracts.

External APIs require different implementations when being consumed from outside. It does matter what environment consumer is using.

The client dictates the channel and the environment within the request.

The client dictates the channel within the request but the environment is inherited from the application.

External APIs require exact implementations when being consumed from outside. It doesn't matter what environment consumer is using.

The client dictates the channel and the environment within the request.

The client dictates the channel within the request but the environment is inherited from the application.

In the implementations below SocialMedia is the general purpose interface. Whether a channel needs a method or not, it has to be inherited at least with an empty body. Also the methods will have to have the same signatures. For that you will have to standardise the arguments as well. Maybe with a general purpose configuration struct. From a design perspective, 2.2 is the best and 1.1 is the worst scenarios to implement. You can adjust these examples as per your needs so they are there just to give you an idea.


1.1

External APIs require different implementations when being consumed from outside. It does matter what environment consumer is using. The client dictates the channel and the environment within the request.


Endpoints

# Image upload
- GET /image/upload?name={facebook|twitter}&env={production|sandbox}
# Message send
- GET /message/send?name={facebook|twitter}&env={production|sandbox}
# Token create
- GET /token/create?name={facebook|twitter}&env={production|sandbox}
# Token refresh
- GET /token/refresh?name={facebook|twitter}&env={production|sandbox}

Structure

├── cmd
│   └── socialise
│       └── main.go
└── internal
    ├── image
    │   └── upload.go
    ├── message
    │   └── send.go
    ├── pkg
    │   └── socialmedia
    │       ├── channel.go
    │       ├── facebook
    │       │   ├── production
    │       │   │   ├── facebook.go
    │       │   │   ├── image.go
    │       │   │   ├── message.go
    │       │   │   └── token.go
    │       │   └── sandbox
    │       │       ├── facebook.go
    │       │       ├── image.go
    │       │       ├── message.go
    │       │       └── token.go
    │       └── twitter
    │           ├── production
    │           │   ├── image.go
    │           │   ├── message.go
    │           │   ├── token.go
    │           │   └── twitter.go
    │           └── sandbox
    │               ├── image.go
    │               ├── message.go
    │               ├── token.go
    │               └── twitter.go
    └── token
        ├── create.go
        └── refresh.go

Files

cmd/socialise/main.go

package main

import (
	"log"
	"net/http"

	"socialise/internal/image"
	"socialise/internal/message"
	"socialise/internal/pkg/socialmedia"
	"socialise/internal/token"

	facebookproduction "socialise/internal/pkg/socialmedia/facebook/production"
	facebooksandbox "socialise/internal/pkg/socialmedia/facebook/sandbox"
	twitterproduction "socialise/internal/pkg/socialmedia/twitter/production"
	twittersandbox "socialise/internal/pkg/socialmedia/twitter/sandbox"
)

func main() {
	// APPLICATION BOOTSTRAP
	// ---------------------

	// Register all social media channels with environments
	sm := socialmedia.New()

	sm.Add(facebookproduction.Name, socialmedia.Production, facebookproduction.New(facebookproduction.Config{
		BaseURI:  "https://api.facebook.com",
		Username: "usr_1",
		Password: "psw_1",
	}))
	sm.Add(facebooksandbox.Name, socialmedia.Sandbox, facebooksandbox.New(facebooksandbox.Config{
		BaseURI:  "https://api-sandbox.facebook.com",
		Username: "usr_2",
		Password: "psw_2",
	}))
	sm.Add(twitterproduction.Name, socialmedia.Production, twitterproduction.New(twitterproduction.Config{
		BaseURI:      "https://api.twitter.com",
		ClientID:     "cid_1",
		ClientSecret: "csr_1",
	}))
	sm.Add(twittersandbox.Name, socialmedia.Sandbox, twittersandbox.New(twittersandbox.Config{
		BaseURI:      "https://api-sandbox.twitter.com",
		ClientID:     "cid_2",
		ClientSecret: "csr_2",
	}))
	//

	// Setup all routes
	imgUpld := image.NewUpload(*sm)
	msgSend := message.NewSend(*sm)
	tokCret := token.NewCreate(*sm)
	tokRefr := token.NewRefresh(*sm)
	//

	// Register HTTP routers
	router := http.NewServeMux()
	router.HandleFunc("/image/upload", imgUpld.Handle)
	router.HandleFunc("/message/send", msgSend.Handle)
	router.HandleFunc("/token/create", tokCret.Handle)
	router.HandleFunc("/token/refresh", tokRefr.Handle)
	//

	// Start HTTP server
	log.Println("http://0.0.0.0:8910")
	server := &http.Server{Addr: ":8910", Handler: router}
	log.Fatalln(server.ListenAndServe())
	//
}

internal/image/upload.go

package image

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Upload struct {
	channel socialmedia.Channel
}

// GET /image/upload?name={facebook|twitter}&env={production|sandbox}
func NewUpload(channel socialmedia.Channel) Upload {
	return Upload{
		channel: channel,
	}
}

func (i Upload) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := i.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.UploadImage()))
}

internal/message/send.go

package message

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Send struct {
	channel socialmedia.Channel
}

// GET /message/send?name={facebook|twitter}&env={production|sandbox}
func NewSend(channel socialmedia.Channel) Send {
	return Send{
		channel: channel,
	}
}

func (m Send) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := m.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.SendMessage()))
}

internal/pkg/socialmedia/channel.go

package socialmedia

import (
	"fmt"
)

type (
	Name string
	Env  string
)

const (
	Production Env = "production"
	Sandbox        = "sandbox"
)

type SocialMedia interface {
	CreateToken() string
	RefreshToken() string
	UploadImage() string
	SendMessage() string
}

type Channel struct {
	channels map[Name]map[Env]SocialMedia
}

func New() *Channel {
	return &Channel{
		channels: make(map[Name]map[Env]SocialMedia),
	}
}

func (m *Channel) Add(name Name, env Env, channel SocialMedia) {
	val, ok := m.channels[name]
	if !ok {
		val = make(map[Env]SocialMedia)
		m.channels[name] = val
	}
	m.channels[name][env] = channel
}

func (m *Channel) Get(name Name, env Env) (SocialMedia, error) {
	val, ok := m.channels[name][env]
	if !ok {
		return nil, fmt.Errorf("invalid channel environment: %s %s", name, env)
	}

	return val, nil
}

internal/pkg/socialmedia/facebook/production/facebook.go

package production

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "facebook"

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/production/image.go

package production

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/production/message.go

package production

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/production/token.go

package production

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/facebook.go

package sandbox

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "facebook"

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/sandbox/image.go

package sandbox

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/message.go

package sandbox

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/token.go

package sandbox

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/twitter/production/image.go

package production

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/message.go

package production

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/token.go

package production

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/twitter.go

package production

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "twitter"

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/pkg/socialmedia/twitter/sandbox/image.go

package sandbox

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/message.go

package sandbox

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/token.go

package sandbox

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/twitter.go

package sandbox

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "twitter"

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/token/create.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Create struct {
	channel socialmedia.Channel
}

// GET /token/create?name={facebook|twitter}&env={production|sandbox}
func NewCreate(channel socialmedia.Channel) Create {
	return Create{
		channel: channel,
	}
}

func (t Create) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := t.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.CreateToken()))
}

internal/token/refresh.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Refresh struct {
	channel socialmedia.Channel
}

// GET /token/refresh?name={facebook|twitter}&env={production|sandbox}
func NewRefresh(channel socialmedia.Channel) Refresh {
	return Refresh{
		channel: channel,
	}
}

func (t Refresh) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := t.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.RefreshToken()))
}

1.2

External APIs require different implementations when being consumed from outside. It does matter what environment consumer is using. The client dictates the channel within the request but the environment is inherited from the application.


Endpoints

# Image upload
- GET /image/upload?name={facebook|twitter}
# Message send
- GET /message/send?name={facebook|twitter}
# Token create
- GET /token/create?name={facebook|twitter}
# Token refresh
- GET /token/refresh?name={facebook|twitter}

Structure

├── cmd
│   └── socialise
│       └── main.go
└── internal
    ├── image
    │   └── upload.go
    ├── message
    │   └── send.go
    ├── pkg
    │   └── socialmedia
    │       ├── channel.go
    │       ├── facebook
    │       │   ├── production
    │       │   │   ├── facebook.go
    │       │   │   ├── image.go
    │       │   │   ├── message.go
    │       │   │   └── token.go
    │       │   └── sandbox
    │       │       ├── facebook.go
    │       │       ├── image.go
    │       │       ├── message.go
    │       │       └── token.go
    │       └── twitter
    │           ├── production
    │           │   ├── image.go
    │           │   ├── message.go
    │           │   ├── token.go
    │           │   └── twitter.go
    │           └── sandbox
    │               ├── image.go
    │               ├── message.go
    │               ├── token.go
    │               └── twitter.go
    └── token
        ├── create.go
        └── refresh.go

Files

cmd/socialise/main.go

package main

import (
	"log"
	"net/http"

	"socialise/internal/image"
	"socialise/internal/message"
	"socialise/internal/pkg/socialmedia"
	"socialise/internal/token"

	facebookproduction "socialise/internal/pkg/socialmedia/facebook/production"
	facebooksandbox "socialise/internal/pkg/socialmedia/facebook/sandbox"
	twitterproduction "socialise/internal/pkg/socialmedia/twitter/production"
	twittersandbox "socialise/internal/pkg/socialmedia/twitter/sandbox"
)

func main() {
	// APPLICATION BOOTSTRAP
	// ---------------------

	// Define application environment
	env := "sandbox"

	// Register all social media channels with environments
	sm := socialmedia.New(socialmedia.Env(env))

	switch socialmedia.Env(env) {
	case socialmedia.Production:
		sm.Add(facebookproduction.Name, socialmedia.Production, facebookproduction.New(facebookproduction.Config{
			BaseURI:  "https://api.facebook.com",
			Username: "usr_1",
			Password: "psw_1",
		}))
		sm.Add(twitterproduction.Name, socialmedia.Production, twitterproduction.New(twitterproduction.Config{
			BaseURI:      "https://api.twitter.com",
			ClientID:     "cid_1",
			ClientSecret: "csr_1",
		}))
	case socialmedia.Sandbox:
		sm.Add(facebooksandbox.Name, socialmedia.Sandbox, facebooksandbox.New(facebooksandbox.Config{
			BaseURI:  "https://api-sandbox.facebook.com",
			Username: "usr_2",
			Password: "psw_2",
		}))

		sm.Add(twittersandbox.Name, socialmedia.Sandbox, twittersandbox.New(twittersandbox.Config{
			BaseURI:      "https://api-sandbox.twitter.com",
			ClientID:     "cid_2",
			ClientSecret: "csr_2",
		}))
	default:
		log.Fatalln("invalid environment:", env)
	}
	//

	// Setup all routes
	imgUpld := image.NewUpload(*sm)
	msgSend := message.NewSend(*sm)
	tokCret := token.NewCreate(*sm)
	tokRefr := token.NewRefresh(*sm)
	//

	// Register HTTP routers
	router := http.NewServeMux()
	router.HandleFunc("/image/upload", imgUpld.Handle)
	router.HandleFunc("/message/send", msgSend.Handle)
	router.HandleFunc("/token/create", tokCret.Handle)
	router.HandleFunc("/token/refresh", tokRefr.Handle)
	//

	// Start HTTP server
	log.Println("http://0.0.0.0:8910")
	server := &http.Server{Addr: ":8910", Handler: router}
	log.Fatalln(server.ListenAndServe())
	//
}

internal/image/upload.go

package image

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Upload struct {
	channel socialmedia.Channel
}

// GET /image/upload?name={facebook|twitter}
func NewUpload(channel socialmedia.Channel) Upload {
	return Upload{
		channel: channel,
	}
}

func (i Upload) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := i.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.UploadImage()))
}

internal/message/send.go

package message

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Send struct {
	channel socialmedia.Channel
}

// GET /message/send?name={facebook|twitter}
func NewSend(channel socialmedia.Channel) Send {
	return Send{
		channel: channel,
	}
}

func (m Send) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := m.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.SendMessage()))
}

internal/pkg/socialmedia/channel.go

package socialmedia

import (
	"fmt"
)

type (
	Name string
	Env  string
)

const (
	Production Env = "production"
	Sandbox        = "sandbox"
)

type SocialMedia interface {
	CreateToken() string
	RefreshToken() string
	UploadImage() string
	SendMessage() string
}

type Channel struct {
	env      Env
	channels map[Name]map[Env]SocialMedia
}

func New(env Env) *Channel {
	return &Channel{
		env:      env,
		channels: make(map[Name]map[Env]SocialMedia),
	}
}

func (m *Channel) Add(name Name, env Env, channel SocialMedia) {
	val, ok := m.channels[name]
	if !ok {
		val = make(map[Env]SocialMedia)
		m.channels[name] = val
	}
	m.channels[name][env] = channel
}

func (m *Channel) Get(name Name) (SocialMedia, error) {
	val, ok := m.channels[name][m.env]
	if !ok {
		return nil, fmt.Errorf("invalid channel environment: %s %s", name, m.env)
	}

	return val, nil
}

internal/pkg/socialmedia/facebook/production/facebook.go

package production

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "facebook"

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/production/image.go

package production

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/production/message.go

package production

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/production/token.go

package production

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/facebook.go

package sandbox

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "facebook"

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/sandbox/image.go

package sandbox

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/message.go

package sandbox

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/token.go

package sandbox

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/twitter/production/image.go

package production

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/message.go

package production

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/token.go

package production

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/twitter.go

package production

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "twitter"

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/pkg/socialmedia/twitter/sandbox/image.go

package sandbox

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/message.go

package sandbox

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/token.go

package sandbox

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/twitter.go

package sandbox

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "twitter"

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/token/create.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Create struct {
	channel socialmedia.Channel
}

// GET /token/create?name={facebook|twitter}
func NewCreate(channel socialmedia.Channel) Create {
	return Create{
		channel: channel,
	}
}

func (t Create) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := t.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.CreateToken()))
}

internal/token/refresh.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Refresh struct {
	channel socialmedia.Channel
}

// GET /token/refresh?name={facebook|twitter}
func NewRefresh(channel socialmedia.Channel) Refresh {
	return Refresh{
		channel: channel,
	}
}

func (t Refresh) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := t.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.RefreshToken()))
}

2.1

External APIs require exact implementations when being consumed from outside. It doesn't matter what environment consumer is using. The client dictates the channel and the environment within the request.


Endpoints

# Image upload
- GET /image/upload?name={facebook|twitter}&env={production|sandbox}
# Message send
- GET /message/send?name={facebook|twitter}&env={production|sandbox}
# Token create
- GET /token/create?name={facebook|twitter}&env={production|sandbox}
# Token refresh
- GET /token/refresh?name={facebook|twitter}&env={production|sandbox}

Structure

├── cmd
│   └── socialise
│       └── main.go
├── go.mod
└── internal
    ├── image
    │   └── upload.go
    ├── message
    │   └── send.go
    ├── pkg
    │   └── socialmedia
    │       ├── channel.go
    │       ├── facebook
    │       │   ├── facebook.go
    │       │   ├── image.go
    │       │   ├── message.go
    │       │   └── token.go
    │       └── twitter
    │           ├── image.go
    │           ├── message.go
    │           ├── token.go
    │           └── twitter.go
    └── token
        ├── create.go
        └── refresh.go

Files

cmd/socialise/main.go

package main

import (
	"log"
	"net/http"

	"socialise/internal/image"
	"socialise/internal/message"
	"socialise/internal/pkg/socialmedia"
	"socialise/internal/pkg/socialmedia/facebook"
	"socialise/internal/pkg/socialmedia/twitter"
	"socialise/internal/token"
)

func main() {
	// APPLICATION BOOTSTRAP
	// ---------------------

	// Register all social media channels
	sm := socialmedia.New()
	sm.Add(facebook.Name, facebook.Production, facebook.New(facebook.Config{
		BaseURI:  "https://api.facebook.com",
		Username: "usr",
		Password: "psw",
	}))
	sm.Add(facebook.Name, facebook.Sandbox, facebook.New(facebook.Config{
		BaseURI:  "https://api-sandbox.facebook.com",
		Username: "usr",
		Password: "psw",
	}))
	sm.Add(twitter.Name, twitter.Production, twitter.New(twitter.Config{
		BaseURI:      "https://api.twitter.com",
		ClientID:     "cid",
		ClientSecret: "csr",
	}))
	sm.Add(twitter.Name, twitter.Staging, twitter.New(twitter.Config{
		BaseURI:      "https://api-staging.twitter.com",
		ClientID:     "cid",
		ClientSecret: "csr",
	}))

	// Setup all routes
	imgUpld := image.NewUpload(*sm)
	msgSend := message.NewSend(*sm)
	tokCret := token.NewCreate(*sm)
	tokRefr := token.NewRefresh(*sm)
	//

	// Register HTTP routers
	router := http.NewServeMux()
	router.HandleFunc("/image/upload", imgUpld.Handle)
	router.HandleFunc("/message/send", msgSend.Handle)
	router.HandleFunc("/token/create", tokCret.Handle)
	router.HandleFunc("/token/refresh", tokRefr.Handle)
	//

	// Start HTTP server
	log.Println("http://0.0.0.0:8910")
	server := &http.Server{Addr: ":8910", Handler: router}
	log.Fatalln(server.ListenAndServe())
	//
}

internal/image/upload.go

package image

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Upload struct {
	channel socialmedia.Channel
}

// GET /image/upload?name={facebook|twitter}&env={production|sandbox|staging}
func NewUpload(channel socialmedia.Channel) Upload {
	return Upload{
		channel: channel,
	}
}

func (i Upload) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := i.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.UploadImage()))
}

internal/message/send.go

package message

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Send struct {
	channel socialmedia.Channel
}

// GET /message/send?name={facebook|twitter}&env={production|sandbox|staging}
func NewSend(channel socialmedia.Channel) Send {
	return Send{
		channel: channel,
	}
}

func (m Send) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := m.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.SendMessage()))
}

internal/pkg/socialmedia/channel.go

package socialmedia

import (
	"fmt"
)

type (
	Name string
	Env  string
)

type SocialMedia interface {
	CreateToken() string
	RefreshToken() string
	UploadImage() string
	SendMessage() string
}

type Channel struct {
	channels map[Name]map[Env]SocialMedia
}

func New() *Channel {
	return &Channel{
		channels: make(map[Name]map[Env]SocialMedia),
	}
}

func (m *Channel) Add(name Name, env Env, channel SocialMedia) {
	val, ok := m.channels[name]
	if !ok {
		val = make(map[Env]SocialMedia)
		m.channels[name] = val
	}
	m.channels[name][env] = channel
}

func (m *Channel) Get(name Name, env Env) (SocialMedia, error) {
	val, ok := m.channels[name][env]
	if !ok {
		return nil, fmt.Errorf("invalid channel environment: %s %s", name, env)
	}

	return val, nil
}

internal/pkg/socialmedia/facebook/facebook.go

package facebook

import (
	"socialise/internal/pkg/socialmedia"
)

const (
	Name       socialmedia.Name = "facebook"
	Production socialmedia.Env  = "production"
	Sandbox                     = "sandbox"
)

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/image.go

package facebook

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/message.go

package facebook

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/token.go

package facebook

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/twitter/image.go

package twitter

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/message.go

package twitter

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/token.go

package twitter

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/twitter.go

package twitter

import (
	"socialise/internal/pkg/socialmedia"
)

const (
	Name       socialmedia.Name = "twitter"
	Production socialmedia.Env  = "production"
	Staging                     = "staging"
)

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/token/create.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Create struct {
	channel socialmedia.Channel
}

// GET /token/create?name={facebook|twitter}&env={production|sandbox|staging}
func NewCreate(channel socialmedia.Channel) Create {
	return Create{
		channel: channel,
	}
}

func (t Create) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := t.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.CreateToken()))
}

internal/token/refresh.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Refresh struct {
	channel socialmedia.Channel
}

// GET /token/refresh?name={facebook|twitter}&env={production|sandbox|staging}
func NewRefresh(channel socialmedia.Channel) Refresh {
	return Refresh{
		channel: channel,
	}
}

func (t Refresh) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := t.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.RefreshToken()))
}

2.2

External APIs require exact implementations when being consumed from outside. It doesn't matter what environment consumer is using. The client dictates the channel within the request but the environment is inherited from the application.


Endpoints

# Image upload
- GET /image/upload?name={facebook|twitter}
# Message send
- GET /message/send?name={facebook|twitter}
# Token create
- GET /token/create?name={facebook|twitter}
# Token refresh
- GET /token/refresh?name={facebook|twitter}

Structure

├── cmd
│   └── socialise
│       └── main.go
├── go.mod
└── internal
    ├── image
    │   └── upload.go
    ├── message
    │   └── send.go
    ├── pkg
    │   └── socialmedia
    │       ├── channel.go
    │       ├── facebook
    │       │   ├── facebook.go
    │       │   ├── image.go
    │       │   ├── message.go
    │       │   └── token.go
    │       └── twitter
    │           ├── image.go
    │           ├── message.go
    │           ├── token.go
    │           └── twitter.go
    └── token
        ├── create.go
        └── refresh.go

Files

cmd/socialise/main.go

package main

import (
	"log"
	"net/http"

	"socialise/internal/image"
	"socialise/internal/message"
	"socialise/internal/pkg/socialmedia"
	"socialise/internal/pkg/socialmedia/facebook"
	"socialise/internal/pkg/socialmedia/twitter"
	"socialise/internal/token"
)

func main() {
	// APPLICATION BOOTSTRAP
	// ---------------------

	// Register all social media channels
	sm := socialmedia.New()
	sm.Add(facebook.Name, facebook.New(facebook.Config{
		BaseURI:  "https://api.facebook.com",
		Username: "usr",
		Password: "psw",
	}))
	sm.Add(twitter.Name, twitter.New(twitter.Config{
		BaseURI:      "https://api.twitter.com",
		ClientID:     "cid",
		ClientSecret: "csr",
	}))

	// Setup all routes
	imgUpld := image.NewUpload(*sm)
	msgSend := message.NewSend(*sm)
	tokCret := token.NewCreate(*sm)
	tokRefr := token.NewRefresh(*sm)
	//

	// Register HTTP routers
	router := http.NewServeMux()
	router.HandleFunc("/image/upload", imgUpld.Handle)
	router.HandleFunc("/message/send", msgSend.Handle)
	router.HandleFunc("/token/create", tokCret.Handle)
	router.HandleFunc("/token/refresh", tokRefr.Handle)
	//

	// Start HTTP server
	log.Println("http://0.0.0.0:8910")
	server := &http.Server{Addr: ":8910", Handler: router}
	log.Fatalln(server.ListenAndServe())
	//
}

internal/image/upload.go

package image

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Upload struct {
	channel socialmedia.Channel
}

// GET /image/upload?name={facebook|twitter}
func NewUpload(channel socialmedia.Channel) Upload {
	return Upload{
		channel: channel,
	}
}

func (i Upload) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := i.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.UploadImage()))
}

internal/message/send.go

package message

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Send struct {
	channel socialmedia.Channel
}

// GET /message/send?name={facebook|twitter}
func NewSend(channel socialmedia.Channel) Send {
	return Send{
		channel: channel,
	}
}

func (m Send) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := m.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.SendMessage()))
}

internal/pkg/socialmedia/channel.go

package socialmedia

import (
	"fmt"
)

type Name string

type SocialMedia interface {
	CreateToken() string
	RefreshToken() string
	UploadImage() string
	SendMessage() string
}

type Channel struct {
	channels map[Name]SocialMedia
}

func New() *Channel {
	return &Channel{
		channels: make(map[Name]SocialMedia),
	}
}

func (m *Channel) Add(name Name, channel SocialMedia) {
	m.channels[name] = channel
}

func (m *Channel) Get(name Name) (SocialMedia, error) {
	val, ok := m.channels[name]
	if !ok {
		return nil, fmt.Errorf("invalid channel: %s", name)
	}

	return val, nil
}

internal/pkg/socialmedia/facebook/facebook.go

package facebook

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "facebook"

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/image.go

package facebook

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/message.go

package facebook

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/token.go

package facebook

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/twitter/image.go

package twitter

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/message.go

package twitter

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/token.go

package twitter

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/twitter.go

package twitter

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "twitter"

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/token/create.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Create struct {
	channel socialmedia.Channel
}

// GET /token/create?name={facebook|twitter}
func NewCreate(channel socialmedia.Channel) Create {
	return Create{
		channel: channel,
	}
}

func (t Create) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := t.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.CreateToken()))
}

internal/token/refresh.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Refresh struct {
	channel socialmedia.Channel
}

// GET /token/refresh?name={facebook|twitter}
func NewRefresh(channel socialmedia.Channel) Refresh {
	return Refresh{
		channel: channel,
	}
}

func (t Refresh) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := t.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.RefreshToken()))
}

© 2014 ‐ 2022 inanzzz.com
Dynamically selecting and running the right implementation in Golang
17/05/2020 - GO

We have two external social media channels and want to consume their APIs. The way we would consume these APIs depends on our client's request. We have to design our application in a way that it satisfies two real life scenarios as listed below.


External APIs require different implementations when being consumed from outside. It does matter what environment consumer is using.

External APIs require exact implementations when being consumed from outside. It doesn't matter what environment consumer is using.

As you can imagine, if you are dealing with an external API that falls into the first scenario, you will highly likely have some duplications in your code or structure as well as some extra logic to deal with. However, if you are dealing with an external API that falls into the second scenario, your life would be very easy. All you would deal with is a configuration parameters.


In this example we are going to see examples for both scenarios. It hasn't finished there though. Both scenarios come with two sub scenarios as listed below.


The client dictates the channel and the environment within the request.

The client dictates the channel within the request but the environment is inherited from the application.

In the case of first scenario, the client sends name and env query parameters. In the case of second scenario, the client sends name query parameter. Based on the scenarios above, our application will pick up the right "channel" and "environment" then hit external API accordingly. It's a kind of strategy design pattern.


Finally these are our examples. We will create token, refresh token, send message and upload image as API contracts.


External APIs require different implementations when being consumed from outside. It does matter what environment consumer is using.

The client dictates the channel and the environment within the request.

The client dictates the channel within the request but the environment is inherited from the application.

External APIs require exact implementations when being consumed from outside. It doesn't matter what environment consumer is using.

The client dictates the channel and the environment within the request.

The client dictates the channel within the request but the environment is inherited from the application.

In the implementations below SocialMedia is the general purpose interface. Whether a channel needs a method or not, it has to be inherited at least with an empty body. Also the methods will have to have the same signatures. For that you will have to standardise the arguments as well. Maybe with a general purpose configuration struct. From a design perspective, 2.2 is the best and 1.1 is the worst scenarios to implement. You can adjust these examples as per your needs so they are there just to give you an idea.


1.1

External APIs require different implementations when being consumed from outside. It does matter what environment consumer is using. The client dictates the channel and the environment within the request.


Endpoints

# Image upload
- GET /image/upload?name={facebook|twitter}&env={production|sandbox}
# Message send
- GET /message/send?name={facebook|twitter}&env={production|sandbox}
# Token create
- GET /token/create?name={facebook|twitter}&env={production|sandbox}
# Token refresh
- GET /token/refresh?name={facebook|twitter}&env={production|sandbox}

Structure

├── cmd
│   └── socialise
│       └── main.go
└── internal
    ├── image
    │   └── upload.go
    ├── message
    │   └── send.go
    ├── pkg
    │   └── socialmedia
    │       ├── channel.go
    │       ├── facebook
    │       │   ├── production
    │       │   │   ├── facebook.go
    │       │   │   ├── image.go
    │       │   │   ├── message.go
    │       │   │   └── token.go
    │       │   └── sandbox
    │       │       ├── facebook.go
    │       │       ├── image.go
    │       │       ├── message.go
    │       │       └── token.go
    │       └── twitter
    │           ├── production
    │           │   ├── image.go
    │           │   ├── message.go
    │           │   ├── token.go
    │           │   └── twitter.go
    │           └── sandbox
    │               ├── image.go
    │               ├── message.go
    │               ├── token.go
    │               └── twitter.go
    └── token
        ├── create.go
        └── refresh.go

Files

cmd/socialise/main.go

package main

import (
	"log"
	"net/http"

	"socialise/internal/image"
	"socialise/internal/message"
	"socialise/internal/pkg/socialmedia"
	"socialise/internal/token"

	facebookproduction "socialise/internal/pkg/socialmedia/facebook/production"
	facebooksandbox "socialise/internal/pkg/socialmedia/facebook/sandbox"
	twitterproduction "socialise/internal/pkg/socialmedia/twitter/production"
	twittersandbox "socialise/internal/pkg/socialmedia/twitter/sandbox"
)

func main() {
	// APPLICATION BOOTSTRAP
	// ---------------------

	// Register all social media channels with environments
	sm := socialmedia.New()

	sm.Add(facebookproduction.Name, socialmedia.Production, facebookproduction.New(facebookproduction.Config{
		BaseURI:  "https://api.facebook.com",
		Username: "usr_1",
		Password: "psw_1",
	}))
	sm.Add(facebooksandbox.Name, socialmedia.Sandbox, facebooksandbox.New(facebooksandbox.Config{
		BaseURI:  "https://api-sandbox.facebook.com",
		Username: "usr_2",
		Password: "psw_2",
	}))
	sm.Add(twitterproduction.Name, socialmedia.Production, twitterproduction.New(twitterproduction.Config{
		BaseURI:      "https://api.twitter.com",
		ClientID:     "cid_1",
		ClientSecret: "csr_1",
	}))
	sm.Add(twittersandbox.Name, socialmedia.Sandbox, twittersandbox.New(twittersandbox.Config{
		BaseURI:      "https://api-sandbox.twitter.com",
		ClientID:     "cid_2",
		ClientSecret: "csr_2",
	}))
	//

	// Setup all routes
	imgUpld := image.NewUpload(*sm)
	msgSend := message.NewSend(*sm)
	tokCret := token.NewCreate(*sm)
	tokRefr := token.NewRefresh(*sm)
	//

	// Register HTTP routers
	router := http.NewServeMux()
	router.HandleFunc("/image/upload", imgUpld.Handle)
	router.HandleFunc("/message/send", msgSend.Handle)
	router.HandleFunc("/token/create", tokCret.Handle)
	router.HandleFunc("/token/refresh", tokRefr.Handle)
	//

	// Start HTTP server
	log.Println("http://0.0.0.0:8910")
	server := &http.Server{Addr: ":8910", Handler: router}
	log.Fatalln(server.ListenAndServe())
	//
}

internal/image/upload.go

package image

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Upload struct {
	channel socialmedia.Channel
}

// GET /image/upload?name={facebook|twitter}&env={production|sandbox}
func NewUpload(channel socialmedia.Channel) Upload {
	return Upload{
		channel: channel,
	}
}

func (i Upload) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := i.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.UploadImage()))
}

internal/message/send.go

package message

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Send struct {
	channel socialmedia.Channel
}

// GET /message/send?name={facebook|twitter}&env={production|sandbox}
func NewSend(channel socialmedia.Channel) Send {
	return Send{
		channel: channel,
	}
}

func (m Send) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := m.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.SendMessage()))
}

internal/pkg/socialmedia/channel.go

package socialmedia

import (
	"fmt"
)

type (
	Name string
	Env  string
)

const (
	Production Env = "production"
	Sandbox        = "sandbox"
)

type SocialMedia interface {
	CreateToken() string
	RefreshToken() string
	UploadImage() string
	SendMessage() string
}

type Channel struct {
	channels map[Name]map[Env]SocialMedia
}

func New() *Channel {
	return &Channel{
		channels: make(map[Name]map[Env]SocialMedia),
	}
}

func (m *Channel) Add(name Name, env Env, channel SocialMedia) {
	val, ok := m.channels[name]
	if !ok {
		val = make(map[Env]SocialMedia)
		m.channels[name] = val
	}
	m.channels[name][env] = channel
}

func (m *Channel) Get(name Name, env Env) (SocialMedia, error) {
	val, ok := m.channels[name][env]
	if !ok {
		return nil, fmt.Errorf("invalid channel environment: %s %s", name, env)
	}

	return val, nil
}

internal/pkg/socialmedia/facebook/production/facebook.go

package production

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "facebook"

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/production/image.go

package production

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/production/message.go

package production

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/production/token.go

package production

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/facebook.go

package sandbox

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "facebook"

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/sandbox/image.go

package sandbox

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/message.go

package sandbox

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/token.go

package sandbox

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/twitter/production/image.go

package production

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/message.go

package production

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/token.go

package production

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/twitter.go

package production

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "twitter"

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/pkg/socialmedia/twitter/sandbox/image.go

package sandbox

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/message.go

package sandbox

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/token.go

package sandbox

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/twitter.go

package sandbox

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "twitter"

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/token/create.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Create struct {
	channel socialmedia.Channel
}

// GET /token/create?name={facebook|twitter}&env={production|sandbox}
func NewCreate(channel socialmedia.Channel) Create {
	return Create{
		channel: channel,
	}
}

func (t Create) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := t.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.CreateToken()))
}

internal/token/refresh.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Refresh struct {
	channel socialmedia.Channel
}

// GET /token/refresh?name={facebook|twitter}&env={production|sandbox}
func NewRefresh(channel socialmedia.Channel) Refresh {
	return Refresh{
		channel: channel,
	}
}

func (t Refresh) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := t.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.RefreshToken()))
}

1.2

External APIs require different implementations when being consumed from outside. It does matter what environment consumer is using. The client dictates the channel within the request but the environment is inherited from the application.


Endpoints

# Image upload
- GET /image/upload?name={facebook|twitter}
# Message send
- GET /message/send?name={facebook|twitter}
# Token create
- GET /token/create?name={facebook|twitter}
# Token refresh
- GET /token/refresh?name={facebook|twitter}

Structure

├── cmd
│   └── socialise
│       └── main.go
└── internal
    ├── image
    │   └── upload.go
    ├── message
    │   └── send.go
    ├── pkg
    │   └── socialmedia
    │       ├── channel.go
    │       ├── facebook
    │       │   ├── production
    │       │   │   ├── facebook.go
    │       │   │   ├── image.go
    │       │   │   ├── message.go
    │       │   │   └── token.go
    │       │   └── sandbox
    │       │       ├── facebook.go
    │       │       ├── image.go
    │       │       ├── message.go
    │       │       └── token.go
    │       └── twitter
    │           ├── production
    │           │   ├── image.go
    │           │   ├── message.go
    │           │   ├── token.go
    │           │   └── twitter.go
    │           └── sandbox
    │               ├── image.go
    │               ├── message.go
    │               ├── token.go
    │               └── twitter.go
    └── token
        ├── create.go
        └── refresh.go

Files

cmd/socialise/main.go

package main

import (
	"log"
	"net/http"

	"socialise/internal/image"
	"socialise/internal/message"
	"socialise/internal/pkg/socialmedia"
	"socialise/internal/token"

	facebookproduction "socialise/internal/pkg/socialmedia/facebook/production"
	facebooksandbox "socialise/internal/pkg/socialmedia/facebook/sandbox"
	twitterproduction "socialise/internal/pkg/socialmedia/twitter/production"
	twittersandbox "socialise/internal/pkg/socialmedia/twitter/sandbox"
)

func main() {
	// APPLICATION BOOTSTRAP
	// ---------------------

	// Define application environment
	env := "sandbox"

	// Register all social media channels with environments
	sm := socialmedia.New(socialmedia.Env(env))

	switch socialmedia.Env(env) {
	case socialmedia.Production:
		sm.Add(facebookproduction.Name, socialmedia.Production, facebookproduction.New(facebookproduction.Config{
			BaseURI:  "https://api.facebook.com",
			Username: "usr_1",
			Password: "psw_1",
		}))
		sm.Add(twitterproduction.Name, socialmedia.Production, twitterproduction.New(twitterproduction.Config{
			BaseURI:      "https://api.twitter.com",
			ClientID:     "cid_1",
			ClientSecret: "csr_1",
		}))
	case socialmedia.Sandbox:
		sm.Add(facebooksandbox.Name, socialmedia.Sandbox, facebooksandbox.New(facebooksandbox.Config{
			BaseURI:  "https://api-sandbox.facebook.com",
			Username: "usr_2",
			Password: "psw_2",
		}))

		sm.Add(twittersandbox.Name, socialmedia.Sandbox, twittersandbox.New(twittersandbox.Config{
			BaseURI:      "https://api-sandbox.twitter.com",
			ClientID:     "cid_2",
			ClientSecret: "csr_2",
		}))
	default:
		log.Fatalln("invalid environment:", env)
	}
	//

	// Setup all routes
	imgUpld := image.NewUpload(*sm)
	msgSend := message.NewSend(*sm)
	tokCret := token.NewCreate(*sm)
	tokRefr := token.NewRefresh(*sm)
	//

	// Register HTTP routers
	router := http.NewServeMux()
	router.HandleFunc("/image/upload", imgUpld.Handle)
	router.HandleFunc("/message/send", msgSend.Handle)
	router.HandleFunc("/token/create", tokCret.Handle)
	router.HandleFunc("/token/refresh", tokRefr.Handle)
	//

	// Start HTTP server
	log.Println("http://0.0.0.0:8910")
	server := &http.Server{Addr: ":8910", Handler: router}
	log.Fatalln(server.ListenAndServe())
	//
}

internal/image/upload.go

package image

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Upload struct {
	channel socialmedia.Channel
}

// GET /image/upload?name={facebook|twitter}
func NewUpload(channel socialmedia.Channel) Upload {
	return Upload{
		channel: channel,
	}
}

func (i Upload) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := i.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.UploadImage()))
}

internal/message/send.go

package message

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Send struct {
	channel socialmedia.Channel
}

// GET /message/send?name={facebook|twitter}
func NewSend(channel socialmedia.Channel) Send {
	return Send{
		channel: channel,
	}
}

func (m Send) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := m.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.SendMessage()))
}

internal/pkg/socialmedia/channel.go

package socialmedia

import (
	"fmt"
)

type (
	Name string
	Env  string
)

const (
	Production Env = "production"
	Sandbox        = "sandbox"
)

type SocialMedia interface {
	CreateToken() string
	RefreshToken() string
	UploadImage() string
	SendMessage() string
}

type Channel struct {
	env      Env
	channels map[Name]map[Env]SocialMedia
}

func New(env Env) *Channel {
	return &Channel{
		env:      env,
		channels: make(map[Name]map[Env]SocialMedia),
	}
}

func (m *Channel) Add(name Name, env Env, channel SocialMedia) {
	val, ok := m.channels[name]
	if !ok {
		val = make(map[Env]SocialMedia)
		m.channels[name] = val
	}
	m.channels[name][env] = channel
}

func (m *Channel) Get(name Name) (SocialMedia, error) {
	val, ok := m.channels[name][m.env]
	if !ok {
		return nil, fmt.Errorf("invalid channel environment: %s %s", name, m.env)
	}

	return val, nil
}

internal/pkg/socialmedia/facebook/production/facebook.go

package production

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "facebook"

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/production/image.go

package production

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/production/message.go

package production

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/production/token.go

package production

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/facebook.go

package sandbox

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "facebook"

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/sandbox/image.go

package sandbox

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/message.go

package sandbox

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/sandbox/token.go

package sandbox

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/twitter/production/image.go

package production

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/message.go

package production

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/token.go

package production

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/production/twitter.go

package production

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "twitter"

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/pkg/socialmedia/twitter/sandbox/image.go

package sandbox

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/message.go

package sandbox

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/token.go

package sandbox

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/sandbox/twitter.go

package sandbox

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "twitter"

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/token/create.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Create struct {
	channel socialmedia.Channel
}

// GET /token/create?name={facebook|twitter}
func NewCreate(channel socialmedia.Channel) Create {
	return Create{
		channel: channel,
	}
}

func (t Create) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := t.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.CreateToken()))
}

internal/token/refresh.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Refresh struct {
	channel socialmedia.Channel
}

// GET /token/refresh?name={facebook|twitter}
func NewRefresh(channel socialmedia.Channel) Refresh {
	return Refresh{
		channel: channel,
	}
}

func (t Refresh) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := t.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.RefreshToken()))
}

2.1

External APIs require exact implementations when being consumed from outside. It doesn't matter what environment consumer is using. The client dictates the channel and the environment within the request.


Endpoints

# Image upload
- GET /image/upload?name={facebook|twitter}&env={production|sandbox}
# Message send
- GET /message/send?name={facebook|twitter}&env={production|sandbox}
# Token create
- GET /token/create?name={facebook|twitter}&env={production|sandbox}
# Token refresh
- GET /token/refresh?name={facebook|twitter}&env={production|sandbox}

Structure

├── cmd
│   └── socialise
│       └── main.go
├── go.mod
└── internal
    ├── image
    │   └── upload.go
    ├── message
    │   └── send.go
    ├── pkg
    │   └── socialmedia
    │       ├── channel.go
    │       ├── facebook
    │       │   ├── facebook.go
    │       │   ├── image.go
    │       │   ├── message.go
    │       │   └── token.go
    │       └── twitter
    │           ├── image.go
    │           ├── message.go
    │           ├── token.go
    │           └── twitter.go
    └── token
        ├── create.go
        └── refresh.go

Files

cmd/socialise/main.go

package main

import (
	"log"
	"net/http"

	"socialise/internal/image"
	"socialise/internal/message"
	"socialise/internal/pkg/socialmedia"
	"socialise/internal/pkg/socialmedia/facebook"
	"socialise/internal/pkg/socialmedia/twitter"
	"socialise/internal/token"
)

func main() {
	// APPLICATION BOOTSTRAP
	// ---------------------

	// Register all social media channels
	sm := socialmedia.New()
	sm.Add(facebook.Name, facebook.Production, facebook.New(facebook.Config{
		BaseURI:  "https://api.facebook.com",
		Username: "usr",
		Password: "psw",
	}))
	sm.Add(facebook.Name, facebook.Sandbox, facebook.New(facebook.Config{
		BaseURI:  "https://api-sandbox.facebook.com",
		Username: "usr",
		Password: "psw",
	}))
	sm.Add(twitter.Name, twitter.Production, twitter.New(twitter.Config{
		BaseURI:      "https://api.twitter.com",
		ClientID:     "cid",
		ClientSecret: "csr",
	}))
	sm.Add(twitter.Name, twitter.Staging, twitter.New(twitter.Config{
		BaseURI:      "https://api-staging.twitter.com",
		ClientID:     "cid",
		ClientSecret: "csr",
	}))

	// Setup all routes
	imgUpld := image.NewUpload(*sm)
	msgSend := message.NewSend(*sm)
	tokCret := token.NewCreate(*sm)
	tokRefr := token.NewRefresh(*sm)
	//

	// Register HTTP routers
	router := http.NewServeMux()
	router.HandleFunc("/image/upload", imgUpld.Handle)
	router.HandleFunc("/message/send", msgSend.Handle)
	router.HandleFunc("/token/create", tokCret.Handle)
	router.HandleFunc("/token/refresh", tokRefr.Handle)
	//

	// Start HTTP server
	log.Println("http://0.0.0.0:8910")
	server := &http.Server{Addr: ":8910", Handler: router}
	log.Fatalln(server.ListenAndServe())
	//
}

internal/image/upload.go

package image

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Upload struct {
	channel socialmedia.Channel
}

// GET /image/upload?name={facebook|twitter}&env={production|sandbox|staging}
func NewUpload(channel socialmedia.Channel) Upload {
	return Upload{
		channel: channel,
	}
}

func (i Upload) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := i.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.UploadImage()))
}

internal/message/send.go

package message

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Send struct {
	channel socialmedia.Channel
}

// GET /message/send?name={facebook|twitter}&env={production|sandbox|staging}
func NewSend(channel socialmedia.Channel) Send {
	return Send{
		channel: channel,
	}
}

func (m Send) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := m.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.SendMessage()))
}

internal/pkg/socialmedia/channel.go

package socialmedia

import (
	"fmt"
)

type (
	Name string
	Env  string
)

type SocialMedia interface {
	CreateToken() string
	RefreshToken() string
	UploadImage() string
	SendMessage() string
}

type Channel struct {
	channels map[Name]map[Env]SocialMedia
}

func New() *Channel {
	return &Channel{
		channels: make(map[Name]map[Env]SocialMedia),
	}
}

func (m *Channel) Add(name Name, env Env, channel SocialMedia) {
	val, ok := m.channels[name]
	if !ok {
		val = make(map[Env]SocialMedia)
		m.channels[name] = val
	}
	m.channels[name][env] = channel
}

func (m *Channel) Get(name Name, env Env) (SocialMedia, error) {
	val, ok := m.channels[name][env]
	if !ok {
		return nil, fmt.Errorf("invalid channel environment: %s %s", name, env)
	}

	return val, nil
}

internal/pkg/socialmedia/facebook/facebook.go

package facebook

import (
	"socialise/internal/pkg/socialmedia"
)

const (
	Name       socialmedia.Name = "facebook"
	Production socialmedia.Env  = "production"
	Sandbox                     = "sandbox"
)

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/image.go

package facebook

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/message.go

package facebook

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/token.go

package facebook

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/twitter/image.go

package twitter

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/message.go

package twitter

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/token.go

package twitter

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/twitter.go

package twitter

import (
	"socialise/internal/pkg/socialmedia"
)

const (
	Name       socialmedia.Name = "twitter"
	Production socialmedia.Env  = "production"
	Staging                     = "staging"
)

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/token/create.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Create struct {
	channel socialmedia.Channel
}

// GET /token/create?name={facebook|twitter}&env={production|sandbox|staging}
func NewCreate(channel socialmedia.Channel) Create {
	return Create{
		channel: channel,
	}
}

func (t Create) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := t.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.CreateToken()))
}

internal/token/refresh.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Refresh struct {
	channel socialmedia.Channel
}

// GET /token/refresh?name={facebook|twitter}&env={production|sandbox|staging}
func NewRefresh(channel socialmedia.Channel) Refresh {
	return Refresh{
		channel: channel,
	}
}

func (t Refresh) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]
	env := r.URL.Query()["env"][0]

	media, err := t.channel.Get(socialmedia.Name(name), socialmedia.Env(env))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.RefreshToken()))
}

2.2

External APIs require exact implementations when being consumed from outside. It doesn't matter what environment consumer is using. The client dictates the channel within the request but the environment is inherited from the application.


Endpoints

# Image upload
- GET /image/upload?name={facebook|twitter}
# Message send
- GET /message/send?name={facebook|twitter}
# Token create
- GET /token/create?name={facebook|twitter}
# Token refresh
- GET /token/refresh?name={facebook|twitter}

Structure

├── cmd
│   └── socialise
│       └── main.go
├── go.mod
└── internal
    ├── image
    │   └── upload.go
    ├── message
    │   └── send.go
    ├── pkg
    │   └── socialmedia
    │       ├── channel.go
    │       ├── facebook
    │       │   ├── facebook.go
    │       │   ├── image.go
    │       │   ├── message.go
    │       │   └── token.go
    │       └── twitter
    │           ├── image.go
    │           ├── message.go
    │           ├── token.go
    │           └── twitter.go
    └── token
        ├── create.go
        └── refresh.go

Files

cmd/socialise/main.go

package main

import (
	"log"
	"net/http"

	"socialise/internal/image"
	"socialise/internal/message"
	"socialise/internal/pkg/socialmedia"
	"socialise/internal/pkg/socialmedia/facebook"
	"socialise/internal/pkg/socialmedia/twitter"
	"socialise/internal/token"
)

func main() {
	// APPLICATION BOOTSTRAP
	// ---------------------

	// Register all social media channels
	sm := socialmedia.New()
	sm.Add(facebook.Name, facebook.New(facebook.Config{
		BaseURI:  "https://api.facebook.com",
		Username: "usr",
		Password: "psw",
	}))
	sm.Add(twitter.Name, twitter.New(twitter.Config{
		BaseURI:      "https://api.twitter.com",
		ClientID:     "cid",
		ClientSecret: "csr",
	}))

	// Setup all routes
	imgUpld := image.NewUpload(*sm)
	msgSend := message.NewSend(*sm)
	tokCret := token.NewCreate(*sm)
	tokRefr := token.NewRefresh(*sm)
	//

	// Register HTTP routers
	router := http.NewServeMux()
	router.HandleFunc("/image/upload", imgUpld.Handle)
	router.HandleFunc("/message/send", msgSend.Handle)
	router.HandleFunc("/token/create", tokCret.Handle)
	router.HandleFunc("/token/refresh", tokRefr.Handle)
	//

	// Start HTTP server
	log.Println("http://0.0.0.0:8910")
	server := &http.Server{Addr: ":8910", Handler: router}
	log.Fatalln(server.ListenAndServe())
	//
}

internal/image/upload.go

package image

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Upload struct {
	channel socialmedia.Channel
}

// GET /image/upload?name={facebook|twitter}
func NewUpload(channel socialmedia.Channel) Upload {
	return Upload{
		channel: channel,
	}
}

func (i Upload) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := i.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.UploadImage()))
}

internal/message/send.go

package message

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Send struct {
	channel socialmedia.Channel
}

// GET /message/send?name={facebook|twitter}
func NewSend(channel socialmedia.Channel) Send {
	return Send{
		channel: channel,
	}
}

func (m Send) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := m.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.SendMessage()))
}

internal/pkg/socialmedia/channel.go

package socialmedia

import (
	"fmt"
)

type Name string

type SocialMedia interface {
	CreateToken() string
	RefreshToken() string
	UploadImage() string
	SendMessage() string
}

type Channel struct {
	channels map[Name]SocialMedia
}

func New() *Channel {
	return &Channel{
		channels: make(map[Name]SocialMedia),
	}
}

func (m *Channel) Add(name Name, channel SocialMedia) {
	m.channels[name] = channel
}

func (m *Channel) Get(name Name) (SocialMedia, error) {
	val, ok := m.channels[name]
	if !ok {
		return nil, fmt.Errorf("invalid channel: %s", name)
	}

	return val, nil
}

internal/pkg/socialmedia/facebook/facebook.go

package facebook

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "facebook"

type Config struct {
	BaseURI  string
	Username string
	Password string
}

type Facebook struct {
	Config
}

func New(config Config) Facebook {
	return Facebook{config}
}

internal/pkg/socialmedia/facebook/image.go

package facebook

func (f Facebook) UploadImage() string {
	return "upload image: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/message.go

package facebook

func (f Facebook) SendMessage() string {
	return "send message: " + f.BaseURI
}

internal/pkg/socialmedia/facebook/token.go

package facebook

func (f Facebook) CreateToken() string {
	return "create token: " + f.BaseURI
}

func (f Facebook) RefreshToken() string {
	return "refresh token: " + f.BaseURI
}

internal/pkg/socialmedia/twitter/image.go

package twitter

func (t Twitter) UploadImage() string {
	return "upload image: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/message.go

package twitter

func (t Twitter) SendMessage() string {
	return "send message: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/token.go

package twitter

func (t Twitter) CreateToken() string {
	return "create token: " + t.BaseURI
}

func (t Twitter) RefreshToken() string {
	return "refresh token: " + t.BaseURI
}

internal/pkg/socialmedia/twitter/twitter.go

package twitter

import (
	"socialise/internal/pkg/socialmedia"
)

const Name socialmedia.Name = "twitter"

type Config struct {
	BaseURI      string
	ClientID     string
	ClientSecret string
}

type Twitter struct {
	Config
}

func New(config Config) Twitter {
	return Twitter{config}
}

internal/token/create.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Create struct {
	channel socialmedia.Channel
}

// GET /token/create?name={facebook|twitter}
func NewCreate(channel socialmedia.Channel) Create {
	return Create{
		channel: channel,
	}
}

func (t Create) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := t.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.CreateToken()))
}

internal/token/refresh.go

package token

import (
	"net/http"

	"socialise/internal/pkg/socialmedia"
)

type Refresh struct {
	channel socialmedia.Channel
}

// GET /token/refresh?name={facebook|twitter}
func NewRefresh(channel socialmedia.Channel) Refresh {
	return Refresh{
		channel: channel,
	}
}

func (t Refresh) Handle(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query()["name"][0]

	media, err := t.channel.Get(socialmedia.Name(name))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(err.Error()))
		return
	}
	_, _ = w.Write([]byte(media.RefreshToken()))
}

© 2014 ‐ 2022 inanzzz.com