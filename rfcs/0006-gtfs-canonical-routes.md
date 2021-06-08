- Feature Name: gtfs_canonical_routes
- Start Date: 2021-06-03
- RFC PR: [mbta/technology-docs#0006](https://github.com/mbta/technology-docs/pull/0006)
- Asana task: [asana link](https://app.asana.com/)
- Status: Proposed

# Summary

This will enable us to publish in our static GTFS feed "canonical" data for our rail routes. We will
do so via a "canonical" service that's never active, but which has representative trips, stop times,
shapes, and route patterns.

# Motivation

MBTA.com users are able to see line diagrams that match real-world state, including physical maps,
even when stations are closed for a long duration.

PIOs are able to create alerts for stations without any scheduled service, and T-Alerts subscribers
are able to stay informed about the status of the station closure.

# Guide-level explanation

GTFS output includes additional, hardcoded data about where rail lines go to and where they can stop
if all stations were open, which is independent of all scheduled service.

This is represented as a service with ID "canonical", which is valid for the entire feed date range,
but which has no active days of the week. The typicality of the service is tagged with a new value
so it can be programmatically identified.

All rail routes have two or more trips on this service, corresponding to the "canonical" patterns of
service in each direction for any branches, as understood by stakeholders. Each such trip has a
route pattern, for which it is the representative trip, and a shape, and the route pattern is
considered "typical" for the route. If current service has trips following the same route pattern,
they will use the canonical route pattern as well.

In short, for consumers of the feed, a route pattern has typicality of `1` if and only if it visits
a set of canonical stops in a canonical order.

# Reference-level explanation

The canonical service appears in the following GTFS files in the following way:

- **calendar.txt** includes a new `service_id` to envelop all of the canonical trips on all the
  routes. `start_date` and `end_date` should reflect the feed's valid start/end dates. All other
  fields should have a value of `0`.
- **calendar_attributes.txt** includes the new `service_id`. `service_schedule_type` should be
  `"Other"`, and `rating_start_date` and `rating_end_date` should reflect the feed's valid start/end
  dates. `service_schedule_typicality` should be `6`, and the
  [docs](https://github.com/mbta/gtfs-documentation/blob/master/reference/gtfs.md) should be updated
  with this new kind of value.
- **route_patterns.txt** includes new route patterns for all routes/directions we want to hardcode.
  `route_pattern_typicality` should be `1`, and `route_pattern_sort_order` should be a less than the
  value for all other route patterns for that given route/direction. For routes with branches,
  you'll need a pattern for each route/direction/branch (eg, for Providence/Stoughton Line, you
  should have two new route patterns in the outbound direction).
- **shapes.txt** can probably use the existing shapes.
- **trips.txt** includes one new trip for each new route pattern. It should reference aforementioned
  route pattern and shapes. It should reference the new `service_id` created above. Each of these
  trips should be the `representative_trip_id` for their respective route patterns.
- **stop_times.txt** includes all stops that trains in the route/direction passes through, even if
  they are temporarily closed. Increment the arrival/departure times by one minute for each
  subsequent stop. For a sample, see the attached GTFS file, the output of [✓ 🧪 🚝 Canonical
  trips/patterns experiment](https://app.asana.com/0/881264583703207/1200210504369250), which
  trialled this for Green Line D and Lowell Line (only in the outbound direction).

This will be done as a new "step" of the Makefile called `canonical`, which runs between `truncate`
and `current`. There will be a new a directory, `input/canonical/` to store any static data the step
uses.

The canonical step will be a Python module that reads in the `truncate` feed, and writes out its own
feed after transforming it. The transformations are as above, and also will update any current trips
to use canonical route patterns if relevant, eliminating redundant route patterns.

For Commuter Rail route patterns, the canonical one will specify every stop as non-flag, non-leave
early stop.

# Drawbacks

This is more complicated than not doing it. Some apps may prefer to treat the "typical" route
pattern as the one that has current service. For example, in the case of a long-term stop closure,
perhaps some apps don't want to show the closed stop at all.

# Rationale and alternatives

We could leave the current route patterns and trips alone, and add canonical route patterns with a
new typicality value of `5`. Their only trips would be their single representative trip. With this
approach, since there's no interaction with the remaining data at all, we would make the `canonical`
feed one that goes into the `merged` step. (Though we'd have to figure out how to determine the
valid feed start and end date.)

# Prior art

Until now we've been doing ad-hoc adjustments to determine the route pattern of routes
[here](https://github.com/mbta/api/blob/master/apps/state/config/config.exs#L145).

# Unresolved questions

The main question is whether current service should be put on canonical route patterns, if
applicable, or not.

# Future possibilities

We could expose this more clearly in the API. In the approach outlined here, the "canonical" route
pattern is simply that with a typicality of `1`. In the alternative approach, the typicality would
be `5`. That's not currently filterable in the API. In addition, the `service` can't be filtered by
its typicality, most relevantly when included with `trips`.

Shoudl we do something similar for ferry and/or bus?
