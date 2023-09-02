+++
title = "wot-discovery design notes"

[taxonomies]
categories = ["Releases"]
tags = ["rust", "wot"]
authors = ["Luca Barbato"]
+++

We are quickly iterating over the [wot-discovery](https://crates.io/crates/wot-discovery) crate.
Here some notes about [Discovery specification][Discovery].

## Discovery and Directory

The [W3C Web of Things](https://www.w3.org/WoT/) is all about to make connected devices (Things)
interoperate and cooperate.

In my opinion the greatest achievement is coming up with the [Thing Description][TD] to make possible
to both describe the status quo and pave the way of a more rational alternative. In WoT jargon the former
is called `brownfield` the latter `greenfield`.
But since being able to describe how to interact with Things is only part of the problems we have additional
facets specified in other documents, one of them id [WoT Discovery][Discovery].

The document is mostly oriented on possible greenfield implementations and in my opinion it tries to bring in
too much at the same time.

Once you come up with a mechanism to discover unknown nodes in your networks one may argue that the problem is
solved, other may argue that it doesn't stop at enumeration and you would like to do more:
- Your network may be partitioned so you would need to export one neightbourhood information to another, e.g.
  You may want to have gateways across different kind of networks (e.g. bluetooth vs wifi) and keep a full
  picture.
- Querying may be important if the whole lot of devices may be too much to deal with at the same time, e.g.
  all the sensors in a big city.

Thus the [WoT Discovery](Discovery) document is conceptually split in two:
- Actual means to Advertise and Discover Things.
- Means to aggregate and query and present the Things discovered.

In wot-rust we split the implementations addressing the two concerns in two crates:
- [wot-discovery][w-ds], a library crate, so far is able to discover HTTP/S Things using mDNS-SD.
- [wot-directory][w-dr], a [wot-serve](https://crates.io/crates/wot-serve) extension.

## Discovery

Of the different [Introduction Mechanisms](https://www.w3.org/TR/wot-discovery/#introduction-mech) detailed,
what is currently implemented is the mDNS-mediated DNS-SD system.

The small API surface exposed boils down to setting up a `Discover` that starts the mDNS discovery system and provides a
[Stream](https://docs.rs/futures/latest/futures/stream/trait.Stream.html) of Things by fetching them using HTTP/HTTPS.

In `0.2.0`, the API supports the [wot-td][w-td] [extension](https://docs.rs/wot-td/latest/wot_td/extend/index.html) system.

Since in order to consume the produced Things it is necessary to have a populated `Thing::base` and right now the
specification is not yet explicit on what trasformations/patching the Directory may do with the `0.3.0` we
decided to simply provide the Thing wrapped in a [Discovered](https://docs.rs/wot-discovery/latest/wot_discovery/struct.Discovered.html)
struct and provide accessors to know the address and port of the Servient that provided the Thing Description.

Here the example that lists all the Things available on the local network:

``` rust
use std::future::ready;

use futures_util::StreamExt;
use serde::{Deserialize, Serialize};
use tracing::{info, trace, warn};
use tracing_subscriber::EnvFilter;
use wot_discovery::{Discovered, Discoverer};
use wot_td::extend::ExtendableThing;

#[derive(Debug, Clone, Serialize, Deserialize)]
struct A {}

impl ExtendableThing for A {
    type InteractionAffordance = ();
    type PropertyAffordance = ();
    type ActionAffordance = ();
    type EventAffordance = ();
    type Form = ();
    type ExpectedResponse = ();
    type DataSchema = ();
    type ObjectSchema = ();
    type ArraySchema = ();
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let filter = EnvFilter::try_from_default_env()
        .or_else(|_| EnvFilter::try_new("info"))
        .unwrap();

    tracing_subscriber::fmt().with_env_filter(filter).init();

    let d = Discoverer::new()?.ext::<A>();

    d.stream()?
        .for_each(|discovered| {
            match discovered {
                Ok(Discovered { thing: t, .. }) => {
                    info!("found {:?} {:?}", t.title, t.id,);
                    trace!("{}", serde_json::to_string_pretty(&t).unwrap());
                }
                Err(e) => warn!("something went wrong {:?}", e),
            }
            ready(())
        })
        .await;

    Ok(())
}
```

## Directory

Right now the repository is empty and as the WoT Discovery 2.0 specification start we'll implement a tiny subset of the Directory and
hopefully see if there is enough consensus to support small query languages along with richer but heavy systems such as [SPARQL][SPQL]
even if [oxygraph](https://crates.io/crates/oxigraph) provides us the building blocks to provide it if there is enough interest.

[Discovery]: https://www.w3.org/TR/wot-discovery
[TD]: https://www.w3.org/TR/wot-thing-description
[w-ds]: https://crates.io/crates/wot-discovery
[w-dr]: https://crates.io/crates/wot-directory
[w-td]: https://crates.io/crates/wot-td
[SPQL]: https://www.w3.org/TR/sparql11-overview
