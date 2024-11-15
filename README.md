# Dynamic Services

A dynamic service injection framework that provides an Injection style
programming model implemented as macros with a supporting API. The macros
are implemented in the
[dynamic_services_derive](https://crates.io/crates/dynamic_services_derive) create.

Note that this package is still in experimental stage.

## Injecting Service Consumers

Service consumers are activated and injected with the service as the service
arrives, meaning that the consumer won't exist if the service(s) that it needs
are not there. Services are dynamic. They can come and go dynamically at runtime.
Multiple services of the same type could co-exist and services have additional metadata
(properties) associated with them.

Once the service appears, the consumer will be created and injected with the service.

### Example consumer

A very simple consumer using an example TidalService.

```
// Import your service API so you can use it once the instance appears
use crate::tidal_service::TidalService;
use dynamic_services::ServiceReference;
use dynamic_services_derive::{DynamicServices, dynamic_services};

// Define your consumer struct with at least these derives
#[derive(DynamicServices, Default)]
pub struct MyConsumer {
  #[inject]
  tidal: Option<ServiceReference<TidalService>>,
}
```
The field annotated with `#[inject]` will be injected by the framework. Right
now only the service reference is injected this way, although a reference to
the actual service is passed to the activator. The `#[inject]` annotation does
not need to be separately imported as it is handled by the `DynamicServices`
derive macro.

Provide an impl for your MyConsumer with the `#[dynamic_services]` annotation.
For now the annotation needs to provide the path to the current impl file as
the path attribute,
e.g. if you would declare `use mypkg::consumer::MyConsumer` then `path` must
be `mypkg::consumer`.

```
#[dynamic_services(path=mypkg::consumer)]
impl MyConsumer {
  #[activator]
  pub fn activate(&self, ts: &TidalService) {
    println!("MyConsumer Activated... {} - {:?}", ts.next_event(), self.tidal);
  }

  #[deactivator]
  pub fn deactivate(&self) {
    println!("MyConsumer Deactivated...");
  }

  #[update]
  pub fn update(&self, field: &str, props: std::collections::BTreeMap<String, String>) {
    println!("MyConsumer Updated.");
    println!("  updated field: {}", field);
    println!("  updated properties: {:?}", props);
  }
}
```

The `#[activator]`, `#[deactivator]` and `#[update]` mark callback methods
which are called when the service registration of the dependent service(s)
change. These are all optional.

* The `#[activator]` method is called when all injected services are present
and injected. It also provides a reference to the injected service object(s).
* The `#[deactivator]` method is called when not all required injected services
are available any more.
* The `#[update]` callback is made when the service registration properties
have changed. The field containing the Service Reference to the updated service
is passed in along with the updated properties of the service.

## Example use from main method
Annotate the main method with the `#[dynamic_services_main]` macro. Note that this
will generate the `register_service()`, `update_service()` and `unregister_service()` methods.

```
use dynamic_services_derive::dynamic_services_main;

#[dynamic_services_main]
fn main() {
  ...
```

Then register services like this:
```
  // Create the service
  let ts = TidalService {};

  // Set some service properties in a map
  let mut props = BTreeMap::new();
  props.insert("foo".to_owned(), "bar".to_owned());

  // Register the service and obtain the service registry.
  let sreg = register_service(Box::new(ts), props.clone());
```

You can update the service properties:
```
  props.insert("hi".to_string(), "ha".to_string());
  update_service(&sreg, props);
```

Finally you can un-register the service:
```
  unregister_service(sreg);
```
