---
title: Golang Quirks
comments: true
toc: true
date: 2026-01-02
icon: fas fa-tags
tags: golang
order: 2
---

# Golang Quirks

> _Disclaimer: The following observations reflect my personal experience with the language. What I find "quirky" may seem standard to others, but I found these features particularly interesting when compared to other language implementations. As I am still exploring this ecosystem, I welcome feedback or corrections if any of my interpretations are inaccurate._

## `http.RoundTripper`
If we require a middleware for HTTP Clients. Then we can use the `RoundTripper` *interface*.

### Example
```go
type AuthTripper struct {
	next     http.RoundTripper
	username string
	password string
}

func (a AuthTripper) RoundTrip(r *http.Request) (*http.Response, error) {
	r.SetBasicAuth(a.username, a.password)
	return a.next.RoundTrip(r)
}

func main() {
	// ...
	client := http.Client{
		next: AuthTripper{
			next:     http.DefaultTransport,
			username: "abc",
			password: "xyz",
		},
	}
	// ...
}
```

## JSON Encoder/Decoder
### Encoder
JSON Encoder adds `\n` for every conversion.

This is documented and designed for stream data. But often overlooked or forgotten during implementation. This overlook can cause encryptions to misbehave and ruin the experience

Why did I need this?
`json.Marshal()` will encode to *UTF-8*, hence the only way to avoid this is to use `json.NewEncoder()` with `SetEscapeHTML(false)`. After doing so, the additional `\n` should be trimmed away to avoid the above mentioned problem.

#### Example
```go
var result Data // Random data object
// ... Data population and processing
var jsonBytes bytes.Buffer
enc := json.NewEncoder(&jsonBytes)
enc.SetEscapeHTML(false)
if err := enc.Encode(result); err != nil {
	return err
}
jsonString := strings.TrimSuffix(jsonBytes.String(), "\n")
```

### Decoder
 
When dealing with a HTTP Request, `http.Request.Body` can be directly passed to `json.NewDecoder(io.Reader)` to make the implementation a bit cleaner by writing less code.
But since Decoder is also designed for stream data, `json.NewDecoder.Decode()` will read the request until it ends up with valid JSON format and ignore the rest of the body.
For example

```json
{"msg": "Hello"} abcdef
```

Normally, this is an invalid JSON, and `json.Unmarshal()` would thrown an error. This is not a bug, but how its intended to work, but can be forgotten or overlooked during implementation.

