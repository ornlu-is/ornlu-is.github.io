---
title: "Go Design Patterns: Functional Options"
date: 2023-07-20T22:19:24+01:00
categories: ["Study Notes"]
description: "How to implement the functional options design pattern in Golang"
draft: false
---

The Functional Options pattern is a rather elegant manner of implementing a Golang struct constructor that allows for custom default values, which means that users of the API we are implementing will only need to specify the struct attribute values that the users deem that shouldn't take their default values. 

For our example, let us consider the very simple use case where we have a package named `person` containing a `Person` struct, which will look like this:

```go
type Person struct {
	ID      int
	Name    string
	Age     int
	Email   string
	Address string
}
```

## Before Functional Options

To initialize such an instance of the aforementioned struct, one would usually implement a function called `New`, which would take a bunch of values as parameters and return the struct with the given values, like so:

```go
func New(id int, name string, age int, email string, address string) *Person {
	return &Person{
		ID:      id,
		Name:    name,
		Age:     age,
		Email:   email,
		Address: address,
	}
}
```

To create a new `Person`, one would have to call that function and pass in every single value:
```go
p := person.New(1, "John Doe", 25, "johndoe@example.com", "Nowhere")
```

This might get cumborsome in the case where there are a lot of attributes and some of them might require more intricate knowledge of the inner workings of the package.

## Functional Options Pattern

Functional options to the rescue. The idea behind this pattern is fairly simple, we will have `New` become a variadic function which will accept any number of arguments of the type `Option`, which we define as:

```go
type Option func(*Person)
```

Then, for every struct attribute that should have a default value, we implement a function of the following form:

```go
func WithAttributeName(attributeValue attributeType) Option {
	return func(p *Person) {
		p.AttributeName = attributeValue
	}
}
```

For our case, let us say we want all attributes except `ID` to have default values. In that case, we'd end up with something like:

```go
func WithName(name string) Option {
	return func(p *Person) {
		p.Name = name
	}
}

func WithAge(age int) Option {
	return func(p *Person) {
		p.Age = age
	}
}

func WithEmail(email string) Option {
	return func(p *Person) {
		p.Email = email
	}
}

func WithAddress(address string) Option {
	return func(p *Person) {
		p.Address = address
	}
}
```

Finally, all we need is adapt our `New` function to handle these other functions as parameters and to set some default values. This is simply boils down to creating the `Person` struct with the default values we want and then looping over and calling the given `Option`s: 

```go
func New(id int, opts ...Option) *Person {
	p := &Person{
		ID:      id,
		Name:    "John Doe",
		Age:     25,
		Email:   "johndoe@example.com",
		Address: "Nowhere",
	}
	for _, opt := range opts {
		opt(p)
	}
	return p
}
```

Note that this method sets all attributes as optional except for the `ID` of the `Person`. Below is an example of initializing `Person` instances with functional options:

```go
func main() {
	unknownPerson := person.New(1)
	aragorn := person.New(
		2,
        person.WithName("Aragorn II"),
		person.WithAddress("Rivendell"),
		person.WithAge(118),
		person.WithEmail("aragorn@mithrilmail.com"),
	)

	fmt.Printf("%+v\n", *unknownPerson)
	fmt.Printf("%+v\n", *aragorn)
}
```

This produces the following output:
```plaintext
{ID:1 Name:John Doe Age:25 Email:johndoe@example.com Address:Nowhere}
{ID:2 Name:Aragorn II Age:118 Email:aragorn@mithrilmail.com Address:Rivendell}
```

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/go_functional_options
{{< /admonition >}}
