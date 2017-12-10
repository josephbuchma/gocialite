# Gocialite
![Travis CI build](https://api.travis-ci.org/danilopolani/gocialite.svg?branch=master)
![Available Drivers](https://img.shields.io/badge/Drivers-5+-orange.svg)
[![GoDoc](https://godoc.org/github.com/danilopolani/gocialite?status.svg)](https://godoc.org/github.com/danilopolani/gocialite)
[![GoReport](https://goreportcard.com/badge/github.com/danilopolani/gocialite)](https://goreportcard.com/report/github.com/danilopolani/gocialite)
![GitHub contributors](https://img.shields.io/github/contributors/danilopolani/gocialite.svg)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

Gocialite is a Socialite inspired package to manage social oAuth authentication without problems.
The idea was born when I discovered that Goth is not so flexible: I was using Revel and it was *impossible* to connect them properly.

## Installation

To install it, just run `go get github.com/danilopolani/gocialite` and include it in your app: `import "github.com/danilopolani/gocialite"`.

## Available drivers

- Amazon
- Bitbucket
- Facebook
- Github
- Google
- LinkedIn
- Slack

## Create new driver

Please see [Contributing page](https://github.com/danilopolani/gocialite/blob/master/CONTRIBUTING.md) to learn how to create new driver and test it.

## Set scopes

**Note**: Gocialite set some default scopes for the user profile, for example for *Facebook* it specify `email` and for *Google* `profile, email`.  
When you use the following method, you don't have to rewrite them. 

Use the `Scopes([]string)` method of your `Gocial` instance. Example:

```
var gocial gocialite.Gocial
gocial.Scopes([]string{"public_repo"})
```

## Set driver

Use the `Driver(string)` method of your `Gocial` instance. Example:

```
var gocial gocialite.Gocial
gocial.Driver("facebook")
```

The driver name will be the provider name in lowercase.

## How to use it

**Note**: All Gocialite methods are chainable.

Declare a "global" variable outside your `main` func:

```go
import (
	...
)

var gocial gocialite.Gocial

func main() {
```

Then create a route to use as redirect bridge, for example `/auth/github`. With this route, the user will be redirected to the provider oAuth login. In this case we use Gin Tonic as router. You have to specify the provider with the `Driver()` method.
Then, with `Scopes()`, you can set a list of scopes as slice of strings. It's optional.  
Finally, with `Redirect()` you can obtain the redirect URL. In this method you have to pass three parameters:

1. Client ID
1. Client Secret
1. Redirect URL

```go
func main() {
	router := gin.Default()

	router.GET("/auth/github", redirectHandler)

	router.Run("127.0.0.1:9090")
}

// Redirect to correct oAuth URL
func redirectHandler(c *gin.Context) {
	authURL := gocial.Driver("github"). // Set provider
		Scopes([]string{"public_repo"}). // Set optional scope(s)
		Redirect( // 
			"xxxxxxxxxxxxxx", // Client ID
			"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx", // Client Secret
			"http://localhost:9090/auth/github/callback", // Redirect URL
		)

	// Redirect with authURL
	c.Redirect(http.StatusFound, authURL) // Redirect with 302 HTTP code
}
```

Now create a callback handler route, where we'll receive the content from the provider.  
In order to validate the oAuth and retrieve the data, you have to invoke the `Handle()` method with two query parameters: `state` and `code`. In your URL, they will look like this: `http://localhost:9090/auth/github/callback?state=xxxxxxxx&code=xxxxxxxx`.  
The `Handle()` method returns an error if there's one or `nil`.  
If there are no errors, in the `User` property of your `gocial` variable, you will find the logged in user information and in the `Token` one, the token info (it's a [oauth2.Token struct](https://godoc.org/golang.org/x/oauth2#Token)). The data of the user - which is a [gocialite.User struct](https://github.com/danilopolani/gocialite/blob/master/structs/user.go) - are the following:

- ID
- FirstName
- LastName
- FullName
- Email
- Avatar (URL)
- Raw (the full JSON returned by the provider)

Note that they can be empty.

```go
func main() {
	router := gin.Default()

	router.GET("/auth/github", redirectHandler)
	router.GET("/auth/github/callback", callbackHandler)

	router.Run("127.0.0.1:9090")
}

// Redirect to correct oAuth URL
// Handle callback of provider
func callbackHandler(c *gin.Context) {
	// Retrieve query params for code and state
	code := c.Query("code")
	state := c.Query("state")
	provider := c.Param("provider")

	// Handle callback and check for errors
	user, err := gocial.Driver(provider).Handle(state, code)
	if err != nil {
		c.Writer.Write([]byte("Error: " + err.Error()))
		return
	}

  // Print in terminal user information
	fmt.Printf("%#v", user)
 
	// If no errors, show a success message
	c.Writer.Write([]byte("Welcome, " + user.FullName))
}
```

Please take a look to [multi provider example](https://github.com/danilopolani/gocialite/wiki/Multi-provider-example) for a full working code with Gin Tonic and variable provider handler.
