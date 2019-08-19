---
title: "Testing Serialization errors in Golang"
date: 2019-08-18T22:13:17+02:00
summary: How do you cover negative scenarios for JSON/XML serialization in Golang using the standard library?. There is a trick that can be used whenever you are using a `float64` type in your `struct`.
catgories: ["golang"]
tags: ["golang"]
highlightjslanguages: ["go"]
---

**How do you cover negative scenarios for JSON/XML serialization in Golang using the standard library?**.

Let's say that you are working with web-services, a RESTFul or SOAP API or you are implementing an HTTP Client. One way or another you will have to Serialize/Deserialize (aka Marshalling/Unmarchalling) JSON or XML. Also let's say you are aiming to have certain level of test coverage or at least cover your critical negative paths/scenarios.

Consider the following trivial function (intentionally omitting any kind of HTTP for simplicity):

```go
package float

import (
	"encoding/json"
)

// ExchangeRate represents exchange rate api response
type ExchangeRate struct {
	USD  float64 `json:"USD"`
	Base string  `json:"base"`
}

func rateAsBytes(rate ExchangeRate) ([]byte, error) {
	bytes, err := json.Marshal(rate)
	if err != nil {
		return nil, err
	}
	return bytes, nil
}
```

A test for the happy path could be written as follows:

```go
package float

import (
	"reflect"
	"testing"
)

func Test_rateAsBytes(t *testing.T) {
	rate := ExchangeRate{Base: "EUR", USD: 10.50}

	expected := []byte(`{"USD":10.5,"base":"EUR"}`)

	got, err := rateAsBytes(rate)

	if err != nil {
		t.Errorf("rateAsBytes() error = %v", err)
		return
	}

	if !reflect.DeepEqual(got, expected) {
		t.Errorf("rateAsBytes() = %v, want %v", string(got), string(expected))
	}
}
```
## The problem

Having the original function 4 lines, the happy path test will cover 3 of them for a total of 75% test coverage. *What do you do if you want to cover more?*, or more importantly how will your application behave?. How do you make the standard library `json` to fail if the compiler checks that your `struct` definition and assignments are valid.

One alternative is to **create an interface around the Marshalling** that defines the function signature `func Marshal(v interface{}) ([]byte, error)`, then you can structure your code to use the concrete implementation and a mock for your tests. **But that is too much work for simply covering a simple `if`**.

## The solution

There is **a trick that can be used whenever you are using a `float64` type in your `struct`**. It relies on the math library, specifically on the function [`math.NaN()`](https://golang.org/pkg/math/#NaN) that returns an IEEE 754 `not-a-number` value. That value can be used for assignments on variables and valid during compile time, yet _the `json` library will fail on encoding during runtime_.

The final version of the test, using also [table driven tests](https://github.com/golang/go/wiki/TableDrivenTests), will look as follows (behold the full coverage):

```go
package float

import (
	"math"
	"reflect"
	"testing"
)

func Test_rateAsBytes(t *testing.T) {
	tests := []struct {
		name    string
		rate    ExchangeRate
		want    []byte
		wantErr bool
	}{
		{
			name:    "success case",
			rate:    ExchangeRate{Base: "EUR", USD: 10.50},
			want:    []byte(`{"USD":10.5,"base":"EUR"}`),
			wantErr: false,
		},
		{
			name:    "failed case",
			rate:    ExchangeRate{Base: "EUR", USD: math.NaN()},
			want:    nil,
			wantErr: true,
		},
	}
	for _, tt := range tests {
		tt := tt
		t.Run(tt.name, func(t *testing.T) {
			// Running tests in parallel :)
			t.Parallel()
			got, err := rateAsBytes(tt.rate)
			if (err != nil) != tt.wantErr {
				t.Errorf("rateAsBytes() error = %v, wantErr %v", err, tt.wantErr)
				return
			}
			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("rateAsBytes() = %v, want %v", string(got), string(tt.want))
			}
		})
	}
}
```

The same ~~trick~~ technique can be used also for XML.

See also:

* [The cover story](https://blog.golang.org/cover).
* [Prefer table driven tests](https://dave.cheney.net/2019/05/07/prefer-table-driven-tests) by Dave Cheney.
* [Golang JSON Package](https://godoc.org/encoding/json)
* [Golang XML Package](https://godoc.org/encoding/xml)