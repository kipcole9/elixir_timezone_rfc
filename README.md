# RFC - Timezone support for Elixir

This document requests comments on a proposal to define timezone support for the [Elixir](https://elixir-lang.org) programming language as [requested](https://elixirforum.com/t/call-for-proposals-time-zone-support-in-elixir/14743) by [Michael Muskala](https://michal.muskala.eu) on June 19th, 2018

## Status

This is an incomplete work in progress design to faciliate conversation and contribution.

## Principles

As stated in the original forum message:

* new functions in the Elixir standard library leveraging the time zone database
* a way to provide the time zone database to the Elixir standard library though a package

In addition we accept the general principle that the API presented should be the simplest possible.

## Structs

Timezone support is construed to mean the support of timezones within the definition of the `DateTime` struct.  Today the definition requires or assumes that the timezone is `Etc/UTC`.  Three meta data fields are expected:

* `std_offset` which is the standard offset for this zone from UTC
* `utc_offset` which is the current offset for this zone from UTC.  It may be different to `std_offset` in the case of daylight saving being in affect for the given zone but its definition is not restricted to this case.

No changes are proposed to the `DateTime` or `NaiveDateTime` structs to ensure backward compatibility.  This has implications for

## Assumptions

The core assumptions underlying this RFC are:

1. Time is measured in the same way for all calendars as a continuous 24-hour period starting at 0:0:0.0 representing midnight.  This is not the canonical representation for calendars such as the [Hebrew calendar](https://en.wikipedia.org/wiki/Zmanim), or the [Hindu calendar](https://en.wikipedia.org/wiki/Hindu_units_of_time) or for any calendar whose day does not start at midnight.

2. The current `DateTime.diff/2` and the proposed `DateTime.diff/3` have no [leap second](https://en.wikipedia.org/wiki/Leap_second) awareness and may therefore provide an inaccurate result.  As of June 2018, the error in difference between a date in January 1972 (when leap seconds were introduced) and a date in June 2018 is no more than 27 seconds.

## Defining the timezone provider module

Since there should be no elixir language dependency on a timezone provider module we need a way to inform the system of which module to be used.  This should be done in such a way that:

1. Current applications compile and run unchanged and produce the same results
2. Providers can be exchanged without change to user code.  Different results may be returned from different provider modules since implementations may vary.  For example, different provider modules may use different ways to express a timezone name.
3. Configuration is available at compile time since

Three options are considered:

1. Add a configuration key to the `config.exs` (and derivative) files.  This has the advantage of familiarity and may be appropriate since the usage is compile-time only.  It also means a timezone provider module can be defined per application which may have some advantages in an umbrella application.  It would require the definition of an elixir language configuration key in `config.exs` which has not been a requirement to this point and is therefore a disadvantage.

2. Add a configuration key in `mix.exs` under the `project/0` function. This has the advantage that it is clear this is a project configuration, not an application configuration.  Since `Mix` is a build tool only (not available at runtime) it also makes clear that this is a build configuration.  It has the disadvantage that this would be an unfamiliar use of the `mix.exs` file.

3. Examine the configured dependencies for a module called `DateTime.TimeZone` and use the first one that defines such a module with the required `@behaviour`.  This has the advantage that no explicit configuration key is required. It has the disadvantage that its a very obscure way to configure the API and has the potential to create dependency order conflict if more than one moduile defines the beahviour as would be the case if, as an example, both `Timex` and `Calendar` are configured.

Of the three options presented, using the `mix.exs` file would seem to align best with the idea of a compile time definition that is elixir language oriented rather than application oriented.  However this is an area where undoubtely alternatives and opinions are important for discussion.  For example:
```elixir

defmodule MyApp.Mixfile do
  use Mix.Project

  def project do
    [
      app: :my_app,
      version: "1.0",
      elixir: "~> 1.9",
      timezone_provider: Timex.TimezoneProvider,
      ...
    ]
  end
```

## Additional types
```elixir
@typedoc """
When applying a timezone or converting a timezone it
is possible that either

* The timezone name is not known to the provided timezone package or
* The time provided cannot be resolved in the timezone requested
"""
@type invalid :: :unknown_zone | :invalid_time_in_zone

@typedoc """
Since there is to be no language dependency on the module providing
timezone information, a module that provides timezone data and
conforms to the `Time.Timezone` behaviour must be supplied.
"""
@type Calendar.timezone_provider :: module()
```

## `DateTime.Timezone` behaviour and default Provider

A timezone provider must conform the `@behaviour` `DateTime.Timezone`  The module `DateTime.Timezone` is also the default timezone provider which, for compatibility reasons, only knows the `Etc/UTC` timezone.

## New functions in `DateTime`

Additional functions are proposed to support a minimum viable API.

```elixir
@doc """
Returns the module name of the configured timezone provider.
"""
@spec DateTime.timezone_provider :: Calendar.timezone_provider()


@doc """
Converts a `DateTime.t()` from one timezone to another.
"""
@spec DateTime.to_timezone(DateTime.t(), Calendar.timezone()) ::
  {:ok, DateTime.t()} | {:error, invalid()}
```

## Modified functions in `DateTime`

Some functions will need modification to support a timezone provider. They are defined to return the same results as the current Elixir 1.6 if no provider is configured or provided as a parameter.

```elixir
@doc """
Converts the given NaiveDateTime to DateTime.

It expects a time zone to put the NaiveDateTime in. Currently it only supports
"Etc/UTC" as time zone.

For this proposal it accepts any timezone that is supported by the timezone provider.

## Notes

* Implementation proposed using a macro.  This allows the timezone module to be resolved at compile
 time. If this was defined as a function then there would be two function calls: one to the proxy
 in `DateTime` then one to the underlying `timezone_provider()`

## Examples

    iex> {:ok, datetime} = DateTime.from_naive(~N[2016-05-24 13:26:08.003], "Etc/UTC")
    iex> datetime
    #DateTime<2016-05-24 13:26:08.003Z>

    iex> {:ok, datetime} = DateTime.from_naive(~N[2016-05-24 00:00:00.000], "Australia/Sydney")
    iex> datetime
    #DateTime<2016-05-24 .......>
"""
@spec DateTime.from_naive(NaiveDateTime.t(), Calendar.timezone(), Calendar.timezone_provider()) ::
  {:ok, DateTime.t} | {:error, invalid}

defmacro from_naive(naive_datetime, timezone_name,
     timezone_provider \\ DateTime.timezone_provider) do
  quote do
    unquote(timezone_provider).from_naive(naive_datetime, timezone_name)
  end
end

```