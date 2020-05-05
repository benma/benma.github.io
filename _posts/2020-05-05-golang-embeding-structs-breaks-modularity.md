---
layout: post
title: "Embedding structs and interfaces break modularity and semantic versioning in Go"
---

Recently I have been pondering a fact that is a bit unsettling. If you maintain a library,
and add an exported method to one of your exported structs, you would expect all your library users
to still be compatible with this API change. In terms of [semantic versioning](https://semver.org/),
this would be a *minor version bump*:

> Given a version number MAJOR.MINOR.PATCH, increment the:
> 1. MAJOR version when you make incompatible API changes,
> 2. MINOR version when you add functionality in a backwards compatible manner, and
> 3. PATCH version when you make backwards compatible bug fixes.

After all, adding new features is usually backwards compatible, and as long as you don't change the
existing API, simply upgrading to the library (and not making any other change) should have no
effect.

This is not true in Go, and in some cases it can lead to real bugs.

# Backwards compatibility

Let's define what we mean by *backwards compatibility*. The [Atlassian REST API
policy](https://developer.atlassian.com/platform/marketplace/atlassian-rest-api-policy/) has a
sensible definition:

`An API is Backwards Compatible if a program written against one version of that API will continue
to work the same way, without modification, against future versions of the API.`

In terms of semver, this means that you can always upgrade a library from say `v1.0.0` to `v1.1.0`,
or from `v5.4.0` to `v5.6.1`, without fear of breakage. As long as you stay on the same major
version.

# Embedding structs breaks modularity

Image you are the maintainer of a library `car`, currently at version `v1.0.0`:

{% highlight Go %}
// car.go, v1.0.0
package car

import (
	"fmt"
)

type Car struct {
	Name string
}

func (car *Car) Drive() {
	fmt.Printf("%s is driving\n", car.Name)
}

func MakeDeLorean() *Car {
	return &Car{
		Name: "DeLorean",
	}
}

{% endhighlight %}

There is a downstream app using it like this, which embeds `Car` and serializes it to
[JSON](https://www.json.org/json-en.html):

{% highlight Go %}
// app, using car.go v1.0.0

import "car"

type ExtendedCar struct {
	*Car
	Weight int
}

func main() {
	myRide := &ExtendedCar{
		Car:    car.MakeDeLorean(),
		Weight: 1230,
	}
	jsonBytes, _ := json.MarshalIndent(myRide, "", "  ")
	fmt.Println(string(jsonBytes))
}
{% endhighlight %}

Running this will output:

{% highlight JSON %}
{
  "Name": "DeLorean",
  "Weight": 1230
}
{% endhighlight %}

Now you decide to add a [`MarshalJSON()`](https://golang.org/pkg/encoding/json/#Marshaler) method to
`Car`. Let's start simple: the JSON marshaler you add will produce the same JSON as before.:

{% highlight Go %}
// car.go, 1.1.0 (<- minor version bump from 1.0.0)

[...]

func (car *Car) MarshalJSON() ([]byte, error) {
	return json.Marshal(map[string]string{"Name": car.Name})
}
{% endhighlight %}

Quiz question: what will the app output, once it upgrades to `car v1.1.0`?

If you are like me, you would expect the output to be the same, and embark on an epic bug search to
figure out why your program broke by just doing a minor library upgrade. The output the app produces
now however is:

{% highlight JSON %}
{
  "Name": "DeLorean"
}
{% endhighlight %}

Note how the `Weight` field is suddenly missing. Since `ExtendedCar` embeds `Car`, it now also
implements `json.Marshaler`, which is used by `json.Marshal()` for serialization.

This is a breaking change, as the app might be using the JSON serialization as input to some
database which enforces the previous schema, send it over the wire in a strictly specified protocol,
or expose it in an API of its own.

Since the app did not change a single line of code except for upgrading the dependency, we must
conclude that the `car` library should actually have been a *major* version bump to `v2.0.0`.

"But wait!", you say. "Obviously I would not add a `MarshalJSON` method that doesn't actually change
the JSON output, like it is done in this contrived example. I would only add it if I wanted to
change the default serialization, and then it would obviously be a breaking change!". This could be
argued, seeing that `json.Marshaler` is in the standard library and `MarshalJSON` is widely used, so
you would expect breaking changes if you change the default JSON serialization. When I was
confronted with a bug of this sort, the library author did not consider it, understandably, as it
can be a bit unintuitive. But fair enough, mistakes happen.

However, `json.Marshaler` is simply an interface, and the same issue can happen with any interface,
even those only defined in the app, which the library author can't know anything about.

# Interface in Go (i.e. duck-typing) breaks modularity

Let's revisit the same example, but using a different interface. Say the app prefers flying over
driving:

{% highlight Go %}
// app, using car.go v1.1.0

import "car"

type ExtendedCar struct {
	*Car
	Weight int
}

func main() {
	myRide := &ExtendedCar{
		Car:    car.MakeDeLorean(),
		Weight: 1230,
	}
	switch v := myRide.(type) {
	case interface{ Fly() }:
		v.Fly()
	case interface{ Drive() }:
		v.Drive()
	}
}
{% endhighlight %}

`myRide` could be of a number of different types defined in the app, which all do or do not
implement the `Fly/Drive` interfaces.

You can already imagine where this is going: if you as a library author think it would be a nice
feature to make the DeLorean fly, you would add:

{% highlight Go %}
// car.go, 1.2.0 (<- minor version bump from 1.1.0)

[...]

func (car *Car) Fly() {
	fmt.Printf("%s is flying\n", car.Name)
	panic("I am a bug!")
}
{% endhighlight %}

Running the app now after upgrading to `v1.2.0` results in a panic:

```
DeLorean is flying
panic: I am a bug!
[...]
```

The app again, by just upgrading the library to the next minor version, shows a breaking change.

Neither did the author of the library `car` expect that simply adding a method to a struct could be
a breaking change for a library user, nor did the library user expect that upgrading a dependency
while staying on the same major version would break their program.

# Conclusion

In [duck-typed](https://en.wikipedia.org/wiki/Duck_typing) languages like Go, adding an exported
method to an exported struct should, strictly speaking, always result in a major version bump, as it
is always potentially a breaking change for downstream users. This applies to many other duck-typed
or dynamic languages, such as Python, JS, Ruby.

This is highly unintuitive, and as far as I can tell, most of the time such additions only result in
minor version bumps.

Duck-typing and semantic versioning seem fundamentally incompatible. Semver is all about avoiding
dependency hell by defining which versions of a library are compatible. In duck-typing however, any
addition can be a breaking change.

In Go, there is an additional pitfall in the semantics of interfaces in combination with embedded
structs. Based on this, I recommend to avoid embedding structs in general, especially for imported
packages, and to always be explicit about which methods and fields you access. This way, additions
to the embedded structs are much less likely to inadvertently change the behavior of your program.

In the above example, this would mean:

{% highlight Go %}
type ExtendedCar struct {
	Car *Car
	Weight int
}
{% endhighlight %}


For package maintainers, I recommend to bump the major version when adding methods that implement
well-known interfaces from the stdlib or from popular non-stdlib packages.
