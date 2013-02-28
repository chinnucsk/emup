Erlang Meetup.com API library.

## Status

Very early days, works but needs help with OTP/Rebar, and the current
API hooks are not comprehensive.

Expect significant changes with no regard for backwards compatibility.

## Dependencies

* OAuth: [erlang-oauth](https://github.com/tim/erlang-oauth)
* JSON: [jsx](https://github.com/talentdeficit/jsx)
* KVC: [kvc](https://github.com/etrepum/kvc.git)

## Building

1. `$ rebar get-deps`
2. `$ rebar compile`

## Authenticating

Currently supports the Meetup.com API key as a parameter. OAuth 1.0a
support may arrive someday.

## Examples

    1> emup:start("<API key>").

    2> lists:foreach(fun(X) -> io:format("~ts~n", [X]) end, emup:render_events("Riak", fun(X) -> emup:simple_event_renderer(with_url, X) end)).
     65  02/25  Cambridge        Discuss "cloud war stories"
            * http://www.meetup.com/Cloudy-Mondays/events/92914182/
     16  02/25  Providence       February Prov Ops
            * http://www.meetup.com/provops/events/101785102/
     28  02/26  San Francisco    Deploying NoSQL Riak on Windows Azure
            * http://www.bayazure.org/events/102178482/
     32  02/26  London           Riak in action: how Temetra use Riak to track utility meter readings
            * http://www.meetup.com/riak-london/events/103646822/
     19  02/27  Radnor           Datomic by Joe Snikeris
            * http://www.meetup.com/Clojadelphia/events/99741942/
     ...

    9> MyGroupsJSON = emup_api:groups(member_of).
    10> emup:group_details(group_id, proplists:get_value(<<"id">>, lists:nth(3, proplists:get_value(<<"results">>, MyGroupsJSON)))).
    [{<<"lon">>,-87.63999938964844},
     {<<"visibility">>,<<"public">>},
     {<<"organizer">>,
      [{<<"name">>,<<"Ray Hightower">>},
       {<<"member_id">>,3576901}]},
     {<<"link">>,<<"http://www.meetup.com/ChicagoRuby/">>},
     {<<"state">>,<<"IL">>},
     {<<"join_mode">>,<<"open">>},
     {<<"who">>,<<"Rubyists">>},
     {<<"country">>,<<"US">>},
     {<<"city">>,<<"Chicago">>},

    55> ChicagoGroups = emup:local_groups(34, "Chicago").
    56> ChicagoEvents = emup:local_events(ChicagoGroups).
    57> emup:check_date(ChicagoGroups, ChicagoEvents, {2013, 03, 18}).
    2013/03/18 17:30 (  62/1109)    Chicago area Hadoop User Group (CHUG)                                      SQL Server PDW w. Polybase:  What, Why, How
    2013/03/18 18:00 (  44/ 786)                             momo-chicago                                                Mobile Game Apps and Sencha Touch

## Notes on development

Paging is surprisingly tricky. There are two different ways in which
the Meetup API indicates that the query should be re-run to collect
more information. I think it depends on the API generation.

`/find/groups` will return `Link` values as HTTP headers:

    Link: <https://api.meetup.com/find/groups?zip=60601&page=50&offset=3>; rel="next"
    Link: <https://api.meetup.com/find/groups?zip=60601&page=50&offset=1>; rel="prev"

`/2/events` will return `next` values in metadata (but no `prev` URL):

    "meta": {
         "link": "https://api.meetup.com/2/events",
         "method": "Events",
         "next": "https://api.meetup.com/2/events?group_id=1709331&status=upcoming&order=time&limited_events=False&desc=false&offset=1&format=json&page=3&fields=",
         "title": "Meetup Events v2",
         "total_count": 13,
         "updated": 1360773015000,
         "url": "https://api.meetup.com/2/events?group_id=1709331&status=upcoming&order=time&limited_events=False&desc=false&offset=0&format=json&page=3&fields="
    },

This generation gap also impacts the interpretation of JSON in
general.  The corresponding hash key to `meta` in the v2 API is
`results`, whose value is a list of entries, while the v3 API returns
just a list of entries with no `meta` or `results` key wrappers.

The new API results don't maintain any concept of a cursor, so if you
have 201 results, the default (and maximum) page size (200) will give
you all but 1, and the next URL that the API returns to grab the last
one will also return 200 results; it's up to the API client to filter
out the duplicates.

**This is a bug and has been filed.**

I have implemented a simple `next_page` function to take the URL and
return more results, but I don't have a good solution for removing
duplicates at the moment.
